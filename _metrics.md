(metrics)=

## Application metrics

Docverse captures **application metrics** to understand how people use the platform — which organizations and projects publish documentation, how often builds and edition updates succeed, and which management operations get exercised. These metrics are a product-analytics signal that complements (and is deliberately distinct from) operational observability, and they guide where the team invests engineering effort next.

### Purpose and philosophy

The purpose of application metrics is to answer **product questions about usage**: how Docverse is adopted, which features matter, and where to prioritize work. Following {sqr}`089`, app metrics exist to characterize *how people use a service* so the team can spot usage patterns and make evidence-based roadmap decisions. They are not an operational debugging tool.

This focus draws a clear line between metrics and the other observability planes Docverse already has. Each plane answers a different question and serves a different consumer:

| Plane                  | Question it answers                                   | Mechanism                          | Primary consumer            |
| ---------------------- | ---------------------------------------------------- | ---------------------------------- | --------------------------- |
| **Application metrics** | How is Docverse *used*? (adoption, feature usage)     | Safir metrics → Kafka → Sasquatch  | SQuaRE (product decisions)  |
| Telemetry              | Is the service *healthy*? (CPU, memory, latency)      | Kubernetes / Prometheus            | Operators                   |
| Structured logs        | What *happened* in this request or job?               | structlog → log aggregation        | Operators (diagnosis)       |
| Error tracking         | What *broke*, with a stack trace?                     | Sentry (already wired in)          | Developers                  |

Application metrics deliberately do **not** duplicate any of these. They are not telemetry (resource utilization is an infrastructure concern), not logs (a log line answers "what happened in this one operation"; a metric answers "how often does this happen across all tenants"), and not Sentry error tracking (Sentry is already integrated for exception capture, as seen in the `sentry_sdk.capture_exception` calls throughout the worker functions). The metrics pipeline answers questions that none of the operational planes can: aggregate, longitudinal product usage sliced by tenant and operation.

A defining characteristic of Docverse shapes *which* users the application can even observe. The "users" visible to the Docverse application are **documentation authors, CI systems, and organization administrators** — the principals who upload builds, manage editions, and configure projects through the API. Documentation **readers** are invisible to the application: they fetch pages from a CDN that serves directly from object storage and never touch Docverse (see {ref}`documentation-hosting`). Application metrics therefore measure the *authoring and publishing* side of the platform. Reader-side analytics is a separate concern handled by a different pipeline (see {ref}`metrics-reader-analytics`).

### The Safir metrics mechanism

Docverse uses SQuaRE's **Safir metrics** system. Application code defines each event as a Pydantic model derived from `safir.metrics.EventPayload`, registers a publisher for it on an `EventManager`, and publishes instances at the moment of interest. Each event is Avro-serialized and published to the application's Kafka topic `lsst.square.metrics.events.docverse` (the default `topic_prefix` `lsst.square.metrics.events` joined with the application name `docverse`). Every event carries its own Avro schema named for the event — for example `build_processed` — and Sasquatch's InfluxDB sink lands each event name as its own measurement, where it can be queried and dashboarded alongside the rest of Rubin's metrics.

Events are **registered in a publisher container** that implements Safir's `EventMaker` interface: an `initialize(manager)` method creates one publisher per event type via `manager.create_publisher(name, PayloadModel)` and assigns it to an instance attribute. Application code then calls `await events.<name>.publish(payload)`.

Two properties of this mechanism shape every schema in the catalog below.

**Automatic metadata.** The `EventManager` mixes a common `EventMetadata` set of fields into every event before serialization: `id` (a UUID for the event), `application` (`docverse`), `timestamp`, and `timestamp_ns` (the InfluxDB time). Developers never write these — the event payload models in this section define *only* the domain-specific fields, and the four metadata fields are added automatically.

**Scalar-only payloads.** Safir validates at publisher-creation time that every payload field serializes to an InfluxDB-compatible Avro scalar: boolean, int/long, float/double, string, enum, or null (and `timedelta`, which Safir serializes as a numeric duration). Nested models, dicts, and lists are rejected. This constraint drives every design choice in the catalog: events are *flat records of scalars*, so rich relationships (e.g. the per-edition results of a build) are reduced to counts and flags rather than structured sub-objects. Where the queue layer records structured detail (the `progress` JSONB of {ref}`queue`), metrics records only the aggregate scalars suitable for time-series analysis.

**Resilience.** The production `EventManager` is configured with `raise_on_error=False` so that a Kafka outage or schema-registry hiccup never propagates into application code. A failed publish is logged and dropped; it cannot fail a build, an edition publish, or an API request. Metrics are best-effort by design — losing a data point is acceptable, breaking a user operation is not.

The pattern below shows one event end to end. The catalog tables that follow define the remaining payloads; each is "fill in the fields" against this same shape.

```{code-block} python
:caption: src/docverse/events.py (illustrative — one of many events)
from datetime import timedelta

from safir.dependencies.metrics import EventMaker
from safir.metrics import EventManager, EventPayload


class BuildProcessed(EventPayload):
    """Payload for the ``build_processed`` event."""

    organization: str
    project: str | None
    success: bool
    object_count: int
    total_size_bytes: int
    editions_updated: int
    editions_skipped: int
    stale_skipped: bool
    elapsed: timedelta


class DocverseEvents(EventMaker):
    """Container for all Docverse metrics event publishers."""

    async def initialize(self, manager: EventManager) -> None:
        self.build_processed = await manager.create_publisher(
            "build_processed", BuildProcessed
        )
        # ... one create_publisher() call per event in the catalog ...


# Later, in the build_processing worker, once events.initialize() has run:
await events.build_processed.publish(
    BuildProcessed(
        organization=org.slug,
        project=project.slug,
        success=True,
        object_count=object_count,
        total_size_bytes=total_size_bytes,
        editions_updated=len(tracking_result.updated),
        editions_skipped=len(tracking_result.skipped),
        stale_skipped=False,
        elapsed=elapsed,
    )
)
```

(metrics-free)=

### Metrics available without instrumentation

Before designing custom events, it is worth cataloging what the platform emits *for free* — generic metrics that Safir and Gafaelfawr provide with no Docverse-specific code. These cover infrastructure health and raw traffic but, as the limitations show, not product usage.

#### Generic Arq queue metrics

Safir's Arq integration emits queue metrics with two small hooks per worker: `initialize_arq_metrics` in the worker's `on_startup`, and `make_on_job_start(queue_name)` composed into `on_job_start`. With these, every job execution publishes an `arq_job_run` event carrying `time_in_queue` (how long the job waited before a worker picked it up) and `queue` (the queue name), and a periodic `publish_queue_stats` call emits `arq_queue_stats` with `num_queued`. Docverse runs three Arq queues — the **default** queue, the dedicated **keeper-sync** queue, and the dedicated **lifecycle-eval** queue (see {ref}`worker-environments`) — and each can carry these generic metrics independently.

These are valuable for **queue health**: backlog depth and scheduling latency per queue. Their limitation for product analytics is that they are dimensioned only by *queue*, not by organization, project, job kind, or outcome. The default queue alone runs `build_processing`, `publish_edition`, and `dashboard_build` jobs; `arq_job_run` cannot tell them apart, and it has no notion of whether a job succeeded or what tenant it served.

#### Gafaelfawr ingress auth metrics

Traffic to the Docverse API passes through a Gafaelfawr-protected ingress (see {ref}`auth`). Gafaelfawr emits its own generic per-request authentication metrics — `auth_user` / `auth_bot` events carrying fields such as `username`, `service`, and `quota` — for every authenticated request through that ingress. This tells us **who authenticates** against Docverse and at what rate.

Its limitation is symmetric to the queue metrics: the events are emitted by Gafaelfawr and carry no Docverse domain context. They know a principal hit *a* Docverse ingress route; they do not know whether the request created a project, uploaded a build, or listed editions, nor which organization or project was involved.

Together these free metrics cover infrastructure health (queue latency and backlog) and raw API traffic (who authenticates), but they cannot answer product questions sliced by tenant, project, or operation. That gap is exactly what the Docverse event catalog fills.

(metrics-events)=

### Docverse event catalog

The events below are the custom instrumentation Docverse adds. Every event also automatically carries the Safir metadata fields (`id`, `application`, `timestamp`, `timestamp_ns`), which are omitted from the tables. All fields use scalar types only, per the constraint above.

#### Shared payload base

Every Docverse event shares two slicing dimensions, captured in a common base class that all event payloads inherit:

```{code-block} python
:caption: DocverseEventBase — inherited by every Docverse event
class DocverseEventBase(EventPayload):
    organization: str
    project: str | None
```

| Field          | Type        | Description                                                                        |
| -------------- | ----------- | --------------------------------------------------------------------------------- |
| `organization` | str         | Organization (tenant) slug. The primary slice for almost every product question.   |
| `project`      | str \| null | Project slug, when the event concerns a specific project. Null for org-wide events. |

The `organization` slug is a bounded, low-cardinality value (Rubin has a small, slowly-growing set of organizations), which makes it safe and useful as the primary group-by dimension in InfluxDB.

#### Build and publishing events

The core events, with the highest product signal: they measure the central workflow of uploading documentation and getting it live.

**`build_uploaded`** — published in the build handler when a client signals upload complete (`PATCH .../builds/:build` with `status: uploaded`; see {ref}`api`). Captures who is publishing and from what provenance.

| Field                 | Type        | Description                                                                 |
| --------------------- | ----------- | -------------------------------------------------------------------------- |
| `organization`        | str         | Tenant slug (from base).                                                    |
| `project`             | str \| null | Project slug (from base).                                                   |
| `uploader`            | str         | Username of the uploading principal.                                        |
| `git_ref`             | str \| null | Git ref (branch or tag) the build was produced from.                        |
| `ci_platform`         | str \| null | CI platform that produced the build (e.g. `github-actions`), when annotated. |
| `github_event_name`   | str \| null | GitHub event that triggered the workflow (e.g. `push`), when annotated.     |
| `has_alternate_name`  | bool        | Whether the build declared an alternate-name deployment variant.            |

**`build_processed`** — published at the end of the `build_processing` worker (see {ref}`queue`). Measures build outcomes and scale.

| Field               | Type      | Description                                                          |
| ------------------- | --------- | ------------------------------------------------------------------- |
| `organization`      | str       | Tenant slug (from base).                                            |
| `project`           | str       | Project slug (from base).                                           |
| `success`           | bool      | Whether processing completed without failure.                       |
| `object_count`      | int       | Number of objects uploaded to the object store.                     |
| `total_size_bytes`  | int       | Total size of the build in bytes.                                   |
| `editions_updated`  | int       | Number of editions the build's tracking rules updated.              |
| `editions_skipped`  | int       | Number of editions skipped (e.g. superseded).                       |
| `stale_skipped`     | bool      | Whether the whole build was skipped as stale (superseded mid-flight). |
| `elapsed`           | timedelta | Wall-clock processing duration.                                     |

**`edition_published`** — published at the terminal transition of the `publish_edition` worker, i.e. when documentation actually goes live on the CDN. This event directly validates Docverse's "instant edition update" goal by recording how long the publish took.

| Field          | Type      | Description                                                          |
| -------------- | --------- | ------------------------------------------------------------------- |
| `organization` | str       | Tenant slug (from base).                                            |
| `project`      | str       | Project slug (from base).                                           |
| `success`      | bool      | Whether the publish completed without failure.                      |
| `edition_kind` | enum      | Kind of edition (`main`, `release`, `draft`, `major`, `minor`, `alternate`). |
| `trigger`      | enum      | What drove the publish: `build`, `manual`, or `migration`.          |
| `elapsed`      | timedelta | Wall-clock publish duration (CDN sync time).                        |

#### Project and edition management events

Lower-volume configuration actions, consolidated into one parameterized event per resource type (see decision **D4** below).

**`project_lifecycle`** — published in the project create/update/delete handlers.

| Field           | Type | Description                                          |
| --------------- | ---- | --------------------------------------------------- |
| `organization`  | str  | Tenant slug (from base).                            |
| `project`       | str  | Project slug (from base).                           |
| `action`        | enum | `created`, `updated`, or `deleted`.                 |
| `github_bound`  | bool | Whether the project is bound to a GitHub repository. |

**`edition_lifecycle`** — published in the edition create/update/delete/rollback handlers.

| Field           | Type | Description                                                              |
| --------------- | ---- | ---------------------------------------------------------------------- |
| `organization`  | str  | Tenant slug (from base).                                                |
| `project`       | str  | Project slug (from base).                                               |
| `action`        | enum | `created`, `updated`, `deleted`, or `rollback`.                          |
| `edition_kind`  | enum | Kind of edition (`main`, `release`, `draft`, `major`, `minor`, `alternate`). |
| `tracking_mode` | enum | Tracking mode (`git_ref`, `lsst_doc`, `semver_*`, `eups_*`, `alternate_git_ref`). |
| `manual`        | bool | Whether a human performed the action vs. auto-creation by tracking rules. |

#### Membership and access events

Org-scoped access-control changes (`project` is always null).

**`membership_changed`** — published in the membership add/remove handlers.

| Field            | Type | Description                                  |
| ---------------- | ---- | ------------------------------------------- |
| `organization`   | str  | Tenant slug (from base).                    |
| `project`        | null | Always null — membership is org-scoped.     |
| `action`         | enum | `added` or `removed`.                       |
| `role`           | enum | `admin`, `reader`, or `uploader`.           |
| `principal_type` | enum | `user` or `group`.                          |

#### Dashboard events

**`dashboard_built`** — published at the terminal transition of the `dashboard_build` worker (see {ref}`dashboards`).

| Field          | Type      | Description                                                   |
| -------------- | --------- | ------------------------------------------------------------ |
| `organization` | str       | Tenant slug (from base).                                     |
| `project`      | str       | Project slug (from base).                                    |
| `success`      | bool      | Whether the dashboard render completed without failure.      |
| `trigger`      | enum      | What drove the render: `build`, `manual`, or `sync`.         |
| `elapsed`      | timedelta | Wall-clock render duration.                                  |

#### Migration and lifecycle events

Time-boxed and operational-adjacent events. The keeper-sync event is most relevant during the LTD-to-Docverse migration window (see {ref}`migration`); the lifecycle event tracks automated reaping.

**`keeper_sync_run_completed`** — published when a keeper-sync run finalizes.

| Field             | Type      | Description                                                       |
| ----------------- | --------- | ---------------------------------------------------------------- |
| `organization`    | str       | Tenant slug (from base).                                         |
| `project`         | str \| null | Project slug, when the run is project-scoped.                  |
| `status`          | enum      | `succeeded`, `partial_failure`, or `failed`.                    |
| `editions_synced` | int       | Number of editions successfully synced from LTD.                |
| `editions_failed` | int       | Number of editions that failed to sync.                         |
| `elapsed`         | timedelta | Wall-clock run duration.                                        |

**`lifecycle_action`** — published from the `lifecycle_eval` and `git_ref_audit` workers when resources are reaped.

| Field          | Type        | Description                                                      |
| -------------- | ----------- | -------------------------------------------------------------- |
| `organization` | str         | Tenant slug (from base).                                        |
| `project`      | str \| null | Project slug, when the action is project-scoped.               |
| `action`       | enum        | `edition_purged`, `build_purged`, or `ref_deleted`.            |
| `count`        | int         | Number of resources affected by this action.                  |

(metrics-reader-analytics)=

### Reader analytics and the page-view boundary

Docverse application metrics **stop at the authoring and publishing boundary**. Page views, unique readers, popular pages, referrers, and session data are *not observable from application metrics* — by design, not as a gap to be closed later.

The reason is architectural: Docverse does not serve documentation pages. Readers fetch content from a CDN (Fastly or Cloudflare) that serves directly from object storage, bypassing the application entirely (see {ref}`documentation-hosting`). A reader loading a page generates no Docverse event, no log line, and no database query. The application simply never sees the request, so no amount of app-metrics instrumentation could capture it.

Reader-side analytics is instead handled by **[Plausible.io](https://plausible.io)**, the privacy-focused analytics platform used across Rubin documentation. Plausible runs as a separate, client-side pipeline embedded in the served pages; it owns page-view analytics for the whole Rubin documentation estate, Docverse-hosted sites included. Keeping reader analytics in Plausible and authoring/publishing analytics in the Safir metrics pipeline is a deliberate separation of concerns consistent with the usage-understanding goals of {sqr}`089`: each pipeline measures the side of the platform it can actually see, with no overlap and no false attempt to reconstruct reader behavior from application events.

### Design decisions and trade-offs

**D1 — Emit domain job-completion events in addition to the free Arq metrics.** The generic `arq_job_run` and `arq_queue_stats` events ({ref}`metrics-free`) are dimensioned only by queue and lack any notion of organization, project, job kind, or outcome. Product questions — "what is the build success rate for organization X?", "how long do edition publishes take for project Y?" — need exactly those slices. Docverse therefore emits `build_processed`, `edition_published`, and `dashboard_built` as domain events *in addition to* the free queue metrics, accepting the small duplication of a per-job event in exchange for the org/project/outcome dimensions the generic metrics cannot provide.

**D2 — The page-view boundary is intentional.** Reader analytics is Plausible.io's responsibility, deliberately outside the Safir application-metrics pipeline ({ref}`metrics-reader-analytics`). Docverse does not attempt to infer reader behavior from CDN logs or to route reads through the application to observe them; doing so would defeat the CDN architecture that makes hosting fast and cheap.

**D3 — Use a `success: bool` field rather than separate success/failure event classes.** Gafaelfawr models some outcomes as distinct event names (e.g. separate authenticated/unauthenticated events). Docverse instead carries a `success` boolean on the single job-completion event for each operation, because the successful and failed payloads share the same shape (same counts, same durations). A single event with a `success` flag makes failure-ratio queries trivial in InfluxDB (`count(success=false) / count(*)`) without unioning two measurements. This is a conscious deviation from the gafaelfawr pattern, justified by the homogeneous payloads.

**D4 — Consolidate low-volume management actions into one event per resource type.** Rather than minting `project_created`, `project_updated`, and `project_deleted` as three event names, Docverse uses a single `project_lifecycle` event with an `action` enum (and likewise `edition_lifecycle`, `membership_changed`). These actions are low-volume and share a payload shape; one parameterized event per resource keeps the schema catalog small and makes "all activity on projects" a single-measurement query, with `action` available as a group-by when verb-level breakdown is wanted.

**D5 — Avoid high-cardinality fields.** InfluxDB performance degrades with unbounded tag cardinality, so the catalog uses only bounded dimensions: `organization`, `project`, `uploader`, and enums. Deliberately *excluded* as unbounded are `edition_slug`, `build_public_id`, git commit SHAs, and `github_run_id` — all effectively unique per occurrence and unsuitable as time-series dimensions. The one git-ref field retained is `git_ref` on `build_uploaded`, where branch-vs-tag analysis has genuine value and the working set of refs per project is modest; if its cardinality proves problematic in practice, it can be reduced to a bounded `ref_type` (`branch` / `tag`) fallback.

**D6 — Initialize the `DocverseEvents` `EventMaker` in both the app factory and the worker startup.** Events fire from both API handlers (e.g. `build_uploaded`, `project_lifecycle`) and background workers (e.g. `build_processed`, `edition_published`). The `EventManager` and the `DocverseEvents` container must therefore be initialized in *both* the FastAPI application factory (its lifespan) and each Arq worker's `on_startup`, and threaded to call sites the same way Docverse already threads structlog and Sentry — through `RequestContext`/`Factory` for handlers and through the worker `ctx` dict for workers. This mirrors existing plumbing rather than introducing a new injection mechanism (see {ref}`code-architecture`).

**D7 — Disambiguate shared-worker flows with low-cardinality `trigger` enums.** Several flows share a worker: `publish_edition` runs for build-driven updates, manual reassignment/rollback, and migration; `dashboard_build` runs for build, manual, and template-sync triggers. Rather than minting a separate event name per flow, the catalog adds a bounded `trigger` enum to the one event (`edition_published`, `dashboard_built`). This keeps the event-name space small while preserving the ability to slice by originating flow — the same philosophy as **D4**, applied to provenance rather than verb.

### Implementation and testing notes

**One `EventMaker`, two initialization sites.** A single `DocverseEvents(EventMaker)` declares every publisher in its `initialize(manager)` method. Per **D6**, it is initialized in both the FastAPI app factory lifespan and each worker's `on_startup`, and reached at call sites through the existing `RequestContext`/`Factory` plumbing (handlers) and the worker `ctx` (workers). Because production runs with `raise_on_error=False`, no call site needs defensive error handling around `publish()`.

**Testing.** The pytest suite uses Safir's `MockEventManager`, whose publishers record published payloads in place of shipping them to Kafka. Tests assert on emitted events with `assert_published`, verifying both that an operation publishes the expected event and that the payload fields are correct — without a running Kafka or schema registry. This matches Docverse's existing reliance on Safir mocks (e.g. `MockArqQueue`; see {ref}`queue`).

**Phasing.** The build and publishing core (`build_uploaded`, `build_processed`, `edition_published`) ships first, since it carries the highest product signal and instruments the central workflow. Management, membership, dashboard, and migration/lifecycle events are added incrementally as their handlers and workers are built out, each following the established `DocverseEvents` + `EventPayload` pattern. The GitHub Action ({ref}`github-action`) is the dominant source of `build_uploaded` events and provides the CI provenance fields the event records.
