(queue)=

## Task queue design

### Design philosophy

Docverse interacts with the job queue through a **backend-agnostic abstraction layer** (see {ref}`queue-backend-protocol`). The implementation uses [Arq](https://arq-docs.helpmanual.io/) via [Safir's ArqQueue](https://safir.lsst.io/) with Redis as the message transport. The queue backend handles delivery, retries, and worker dispatch. Orchestration and parallelism within a job are handled by Docverse's service layer using standard Python asyncio. This minimizes coupling to any specific queue technology and keeps the business logic testable with plain async functions.

Docverse uses two complementary job-composition patterns:

- **Single self-contained jobs.** A user-facing operation that triggers a bounded unit of work runs as one background job whose worker function calls through the service layer. Where steps are independent, the service layer uses `asyncio.gather()` to parallelize them within the job.
- **Fan-out jobs.** Operations that spread across many independent resources — processing a build that updates several editions, sweeping every organization for lifecycle violations, backfilling an entire LTD instance — run as a small **parent/dispatcher** job that enqueues one **child job per resource**. Each child is independently retryable, holds its own advisory lock, and records its own progress. A parent *run* row aggregates the children's outcomes. This pattern keeps individual jobs small and bounded, isolates per-resource failures, and lets the queue backend schedule the children across workers. Fan-out replaced the earlier "one big job that does everything with `asyncio.gather()` internally" model: for example, `build_processing` now enqueues a separate `publish_edition` child per affected edition instead of publishing them all inline (see {ref}`job-types`).

(worker-pools)=

### Worker pools and queues

Docverse runs **three independent Arq worker pools**, each bound to its own Redis queue and tagged with its own Sentry component. Splitting the workers across queues isolates workloads so a burst on one cannot starve the others: a noisy LTD backfill cannot delay a user's build, and a slow lifecycle sweep cannot delay an edition publish.

| Pool (`WorkerSettings`)        | Queue name                  | Sentry component        | Jobs hosted                                                                                                                                              |
| ------------------------------ | --------------------------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Default**                    | `docverse:queue`            | `worker`                | `build_processing`, `publish_edition`, `dashboard_build`, `dashboard_sync`, `project_github_resolve`, `ping`                                              |
| **Keeper-sync**                | `docverse:sync-queue`       | `worker-keeper-sync`    | `keeper_sync_run_discovery`, `keeper_sync_project`, `keeper_sync_reaper`, `keeper_sync_tier_main`, `keeper_sync_tier_discovery`, `keeper_sync_tier_other` |
| **Lifecycle**                  | `docverse:lifecycle-queue`  | `worker-lifecycle-eval` | `lifecycle_eval_dispatcher`, `lifecycle_eval`, `git_ref_audit_discovery`, `git_ref_audit`, `lifecycle_reaper`, and the run-less reaper backstops          |

The **default** pool carries the latency-sensitive user-facing work (build processing, edition publishing, dashboard rendering). The **keeper-sync** pool is reserved for the LTD migration so a large backfill cannot compete with live publishing (see {ref}`migration`). The **lifecycle** pool carries the periodic maintenance fan-outs (`lifecycle_eval`, `git_ref_audit`) plus, as a deliberate placement, the cron-driven **reaper backstops** for the default-pool kinds — reaper sweeps are light and infrequent, and hosting them here keeps them off the latency-sensitive default pool. (The pool keeps its `lifecycle` lineage in code even though it now hosts more than lifecycle work; a rename is tracked separately.)

All three pools share the same startup/shutdown hooks and the same `WorkerFactoryBuilder`, so every worker — regardless of queue — sees one consistent dependency graph.

### QueueJob table

Docverse maintains its own `queue_jobs` table in Postgres as the single source of truth for job state and progress. This table serves the user-facing queue API, operator dashboards, and internal coordination. The queue backend's internal state is not queried for status — Docverse treats the backend as a delivery mechanism only. See {ref}`table-queue-job` in the database schema section for the column reference within the full schema.

| Column                  | Type                          | Description                                                                                                                                  |
| ----------------------- | ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                    | int                           | Internal PK                                                                                                                                  |
| `public_id`             | int                           | Crockford Base32 serialized in API                                                                                                           |
| `backend_job_id`        | str (nullable)                | Reference to the queue backend's job ID (e.g., Arq UUID)                                                                                     |
| `kind`                  | str                           | Job kind — see the enum below                                                                                                                |
| `status`                | enum                          | `queued`, `in_progress`, `completed`, `completed_with_errors`, `failed`, `cancelled`                                                         |
| `phase`                 | str (nullable)                | Current phase, job-specific (e.g., `unpacking`, `uploading`, `edition_tracking`, `publishing`, `rendering`, `complete`)                      |
| `org_id`                | int                           | Owning org (for operator filtering)                                                                                                          |
| `project_id`            | int (nullable)                | Set for build / publish / dashboard jobs                                                                                                     |
| `build_id`              | int (nullable)                | Set for build processing jobs                                                                                                                |
| `edition_id`            | FK → Edition (nullable)       | Set for `publish_edition` jobs                                                                                                               |
| `keeper_sync_run_id`    | FK → KeeperSyncRun (nullable) | Set for jobs belonging to a keeper-sync run (`ON DELETE SET NULL`)                                                                           |
| `lifecycle_eval_run_id` | FK → LifecycleEvalRun (nullable) | Set for the per-org children of a lifecycle-eval run (`ON DELETE SET NULL`)                                                               |
| `git_ref_audit_run_id`  | FK → GitRefAuditRun (nullable) | Set for the per-org children of a git-ref-audit run (`ON DELETE SET NULL`)                                                                  |
| `subject_label`         | str (nullable)                | Operator-readable subject for fan-out children (the LTD slug for `keeper_sync_project`; the org slug for `lifecycle_eval` / `git_ref_audit`) |
| `progress`              | JSONB (nullable)              | Structured progress data, phase-specific                                                                                                     |
| `errors`                | JSONB (nullable)              | Collected error details                                                                                                                      |
| `date_created`          | datetime                      | When enqueued                                                                                                                                |
| `date_started`          | datetime (nullable)           | When a worker picked it up                                                                                                                   |
| `date_completed`        | datetime (nullable)           | When finished                                                                                                                                |

The `kind` values that get a tracked `queue_jobs` row are:

`build_processing`, `publish_edition`, `dashboard_build`, `dashboard_sync`, `keeper_sync_run_discovery`, `keeper_sync_project`, `lifecycle_eval`, `git_ref_audit`.

The `JobKind` enum also retains the legacy value `edition_update` (the former name for what is now `publish_edition`; see {ref}`job-publish-edition`) and reserves `purgatory_cleanup` and `credential_reencrypt` for **planned** periodic jobs not yet implemented (see {ref}`planned-periodic-jobs`). The orchestration functions — the keeper-sync tier crons, the `lifecycle_eval_dispatcher`, the `git_ref_audit_discovery` tick, the reapers, plus the lightweight `project_github_resolve` and `ping` jobs — are Arq functions that do **not** create `queue_jobs` rows; their effect is the *child* jobs they enqueue and the *run* rows they manage.

#### Run tables

Fan-out subsystems track an overall pass with a dedicated **run** row, and attribute each child `queue_jobs` row to it through the matching nullable FK above:

- `keeper_sync_runs` — one row per operator-triggered LTD backfill (see {ref}`job-keeper-sync`).
- `lifecycle_eval_runs` — one row per hourly lifecycle-eval dispatcher tick.
- `git_ref_audit_runs` — one row per daily git-ref-audit discovery tick.

Run rows deliberately do **not** denormalize per-child counters. Progress is derived on read by aggregating the child `queue_jobs` rows filtered on the run FK (`GROUP BY status`), so there is exactly one source of truth for child state and no counter to keep in sync. A run finalizes — transitioning to `succeeded`, `partial_failure`, or `failed` — once all of its children have reached a terminal status (see {ref}`reaper-pattern`).

(queue-backend-protocol)=

### Queue backend abstraction

The queue backend is accessed through a protocol interface, following the same hexagonal architecture pattern as the object store and CDN abstractions. This keeps the service layer decoupled from any specific queue technology and allows backend swaps without disrupting application logic.

#### Protocol definition

```{code-block} python
from typing import Protocol


class QueueBackend(Protocol):
    """Protocol for queue backend implementations."""

    async def enqueue(
        self,
        job_type: str,
        payload: dict,
        *,
        queue_name: str = "default",
    ) -> str | None:
        """Enqueue a job for background processing.

        Returns the backend-assigned job ID (str), or None if the
        backend does not assign IDs synchronously.
        """
        ...

    async def get_job_metadata(
        self, backend_job_id: str
    ) -> dict | None:
        """Retrieve metadata about a job from the backend.

        Returns backend-specific metadata (e.g., status, result),
        or None if the job is not found. Used for diagnostics only —
        the QueueJob table is the authoritative state store.
        """
        ...

    async def get_job_result(
        self, backend_job_id: str
    ) -> object | None:
        """Retrieve the result of a completed job.

        Returns the job result, or None if not available.
        """
        ...
```

#### Implementations

**`ArqQueueBackend`** wraps Safir's `ArqQueue` for production use. Arq uses UUID strings as job IDs, which are stored in the `backend_job_id` column of the `queue_jobs` table. The `enqueue` call accepts a `queue_name`, so the same backend can dispatch to any of the three queues ({ref}`worker-pools`). The worker functions are standard async functions that receive the job payload and call through the service layer.

**`MockQueueBackend`** wraps Safir's `MockArqQueue` for testing. Jobs are executed in-process, making tests deterministic without requiring a running Redis instance.

Both implementations are constructed by the factory and injected into services, consistent with Docverse's dependency injection pattern.

#### Infrastructure

Arq requires a **Redis** instance as its message broker. In Phalanx deployments, Redis is a standard in-cluster service. The `queue_jobs` Postgres table remains the authoritative state store — Redis holds only transient message data. If Redis state is lost, in-flight jobs can be re-enqueued from `queue_jobs` records with `status = 'queued'`, and the reapers ({ref}`reaper-pattern`) finalize any row whose backend job vanished.

### Progress tracking

Each worker writes progress to its own `queue_jobs` row at phase transitions and at points where granular tracking is useful. These are lightweight single-row UPDATEs through the `QueueJobStore`.

#### Phase transitions

At each phase boundary, the worker updates the `phase` column and writes phase-appropriate `progress` JSONB. For example, `build_processing` walks `unpacking → uploading → edition_tracking → complete`:

```python
await queue_job_store.update_phase(
    queue_job_id, "unpacking",
    progress={"message": "Unpacking build into object store"},
)
# ... unpack tarball, upload objects, record inventory ...

await queue_job_store.update_phase(
    queue_job_id, "edition_tracking",
    progress={"message": "Evaluating edition tracking rules"},
)
tracking_result = await tracking_service.track_build(build)

# Fan out one publish_edition child per affected edition
publish_jobs = await _enqueue_publish_jobs(tracking_result, ...)

await queue_job_store.update_phase(queue_job_id, "complete", progress={...})
await queue_job_store.complete(queue_job_id, has_errors=has_errors)
```

#### Build processing progress shape

Because edition publishing is now fanned out to child jobs, the `build_processing` row's `progress` JSONB records the *outcome of tracking* and a *manifest of the children it spawned*, rather than per-edition publish state:

```json
{
  "message": "Build processing complete",
  "object_count": 1247,
  "total_size_bytes": 184320512,
  "editions_updated": [{ "slug": "__main", "action": "updated" }],
  "editions_skipped": [{ "slug": "v2.x" }],
  "publish_jobs": [
    { "edition_slug": "__main", "publish_queue_job_public_id": "01HQ-3KBR-T5GN-8W" }
  ]
}
```

Each entry in `publish_jobs` points at an independent `publish_edition` child job, whose own `queue_jobs` row carries the publish phase, the CDN sync result, and any failure detail. A superseded build instead records `{"stale_skipped": true, "latest_build_id": …}` and completes without doing any work (see {ref}`cross-job-serialization`). Other jobs use simpler shapes — e.g. `publish_edition` carries `{"message": "Publishing edition"}` during its `publishing` phase, and `dashboard_build` walks `rendering → uploading → complete`.

#### Aggregating fan-out runs

For fan-out subsystems, operators do not read a single job's progress — they read the *run*. The run's status (and a queued/succeeded/failed breakdown) is computed by aggregating the child `queue_jobs` rows on the run FK:

```sql
SELECT status, count(*)
FROM queue_jobs
WHERE keeper_sync_run_id = :run_id
GROUP BY status
```

queued + in-progress children are *pending*, `completed` children *succeeded*, and everything else *failed*; the run finalizes once nothing is pending (see {ref}`reaper-pattern`).

(cross-job-serialization)=

### Cross-job serialization

Several background jobs can race on the same project's resources. Two rapid build uploads from the same branch can produce two `build_processing` jobs for the same `(project, git_ref)`. A `publish_edition` job and another `publish_edition` job (one build-driven, one a manual reassignment) can both try to update the same edition's pointer and per-edition metadata JSON. A `dashboard_build` job triggered by a build and another triggered by a template sync can both try to write the same project's dashboard files. Without coordination these concurrent mutations could interleave KV writes, tear dashboard HTML, or write inconsistent metadata JSON.

Docverse prevents this with **Postgres advisory locks**, combined with a **stale-build supersession guard** in build processing.

#### Lock identifiers

Each lock is a single 64-bit advisory-lock id computed by `compute_lock_id(lock_class, **parts)`: the **high 16 bits** encode a `LockClass` (so a project-level lock can never collide with an edition-level lock that happens to hash to the same value), and the **low 48 bits** are a `blake2b` digest of the resource tuple. blake2b is used rather than Python's built-in `hash()` because it is deterministic across interpreter restarts and worker replicas. There are four lock classes:

| Lock class           | Keyed on                          | Held by                | Serializes                                                            |
| -------------------- | --------------------------------- | ---------------------- | -------------------------------------------------------------------- |
| `BUILD_PROCESSING`   | `(org, project, git_ref)`         | `build_processing`     | Builds for the same branch/tag — and their supersession check        |
| `EDITION_UPDATE`     | `(org, project, edition)`         | `publish_edition`      | An edition's pointer (KV mapping) and its per-edition metadata JSON   |
| `PROJECT`            | `(org, project)`                  | `dashboard_build`      | A project's dashboard render (`__dashboard.html`, `__404.html`, switcher JSON) |
| `DASHBOARD_TEMPLATE` | `(owner, repo, ref, root_path)`   | `dashboard_sync`       | The ETag-compare-and-upsert of a shared dashboard-template content row |

Locks are acquired through the `LockService`, which wraps the `pg_advisory_lock` / `pg_advisory_unlock` pair in an async context manager so the lock scope is visually explicit and released on every exit path:

```python
lock_key = LockKey.for_edition_update(
    org_id=org_id, project_id=project.id, edition_id=edition_id
)
async with lock_service.acquire(lock_key):
    # ... publish the edition under the lock ...
```

`pg_advisory_lock()` blocks (rather than failing) if another session holds the same id, so a competing job simply waits its turn — no job is rejected or fails due to contention.

#### Stale-build supersession guard

`build_processing` acquires the `BUILD_PROCESSING` lock for `(org, project, git_ref)` *before* doing any tarball work, then checks whether a newer build exists for the same branch/tag. If one does, this build is marked **stale-skipped** (its `queue_jobs` row completes with `progress.stale_skipped = true`) and no objects are uploaded or editions touched. Because the check runs inside the lock, two concurrent uploads of the same branch can never both proceed: only the newest build does work; any older build observes a higher latest id and bows out. This guarantees an edition never regresses to an older build of the same ref, regardless of the order in which competing jobs acquire the lock.

#### Why this works

- **No concurrent mutation.** The `EDITION_UPDATE` lock serializes all updates to a given edition, whether the `publish_edition` job was enqueued by build processing, a keeper-sync run, or a manual reassignment/rollback. The `PROJECT` lock serializes every dashboard render for a project, whatever triggered it.
- **No failures.** `pg_advisory_lock()` blocks until the lock is available — the job waits rather than failing.
- **Correct final state.** Stale builds are discarded by the supersession guard; dashboard renders are deterministic from current database state, so the last render to complete is always correct.
- **Independent locks parallelize.** Different editions hash to different `EDITION_UPDATE` ids, so the `publish_edition` children fanned out by one build run concurrently on separate workers without contending; only updates to the *same* edition serialize. Likewise, dashboard renders for different projects acquire independent `PROJECT` locks.

#### Connection impact

An advisory lock holds a database session open for the duration of the locked operation. For an edition publish (~2 seconds for the KV write + cache purge) this is negligible. The `BUILD_PROCESSING` lock is held across the (longer) unpack-and-upload phase, but at most a few builds per `(project, git_ref)` are ever in flight at once. The `PROJECT` dashboard lock is held only for the sub-second write of the project-wide files.

### Operator queries

The `queue_jobs` table provides a single place for operators to understand system state across all pools:

- **Backlog depth**: `SELECT count(*), kind FROM queue_jobs WHERE status = 'queued' GROUP BY kind`
- **Active work**: `SELECT * FROM queue_jobs WHERE status = 'in_progress'` — shows what every worker is doing, which phase each job is in, and (via `subject_label`) which edition, project, or org it serves
- **Run progress**: `SELECT status, count(*) FROM queue_jobs WHERE keeper_sync_run_id = :run_id GROUP BY status` — the live breakdown of a keeper-sync backfill (and likewise for `lifecycle_eval_run_id` / `git_ref_audit_run_id`)
- **Error rates**: `SELECT count(*) FROM queue_jobs WHERE status IN ('failed', 'completed_with_errors') AND date_completed > now() - interval '1 hour'`
- **Per-org throughput**: `SELECT org_id, count(*) FROM queue_jobs WHERE status = 'completed' AND date_completed > now() - interval '1 hour' GROUP BY org_id`
- **Slow / stuck jobs**: `SELECT * FROM queue_jobs WHERE status = 'in_progress' AND date_started < now() - interval '10 minutes'` — candidates the reapers will eventually finalize ({ref}`reaper-pattern`)

(job-types)=

### Job types

The catalog below is grouped by worker pool. Within each pool, fan-out subsystems are described as a parent/child family.

#### Default pool

##### Build processing (`build_processing`)

Triggered when a client signals upload complete (`PATCH .../builds/:build` with `status: uploaded`). This is the primary job type. Under the `BUILD_PROCESSING` lock for `(org, project, git_ref)`, the worker:

1. **Supersession guard** — skip and mark stale if a newer build exists for the same ref ({ref}`cross-job-serialization`).
2. **Unpack and upload** (`unpacking` → `uploading`) — download the staged tarball, unpack it, upload its objects to the object store under `__builds/{build_id}/` (up to 50 concurrent uploads), record the object count and total size, transition the build to `completed`, and delete the staging tarball.
3. **Evaluate tracking rules** (`edition_tracking`) — determine which editions should update for this build's git ref, auto-creating editions where tracking rules call for it. Tracking failures are logged but do not fail the build.
4. **Fan out** — enqueue one `publish_edition` child per updated edition, recording the child job IDs in `progress.publish_jobs`.
5. **Complete** — mark the `queue_jobs` row `completed` (or `completed_with_errors` if tracking failed).

The build job no longer renders the dashboard itself. The dashboard is reached transitively: each `publish_edition` child enqueues a `dashboard_build` on success (deduplicated per project; see below).

(job-publish-edition)=

##### Edition publishing (`publish_edition`)

Syncs a **single** edition's current build to its organization's CDN. This is the worker formerly called `edition_update`; it was renamed and made independently retryable — it resolves all CDN configuration from the database so a retry needs no external context. Under the `EDITION_UPDATE` lock for the edition, it marks the edition and its `EditionBuildHistory` entry `publishing`, performs the CDN sync (KV pointer write or copy-mode object copy + cache purge), and on success marks them `published` and enqueues a `dashboard_build` for the project.

`publish_edition` jobs are produced by several flows — the `build_processing` fan-out, a `keeper_sync_project` sync (migration), and manual edition reassignment or rollback through `EditionService`. A child enqueued by a keeper-sync run carries `keeper_sync_run_id`, so its terminal transition rolls up into the run's progress and can finalize it.

##### Dashboard build (`dashboard_build`)

Renders **one project's** dashboard outputs — the dashboard HTML, the pydata-sphinx-theme switcher JSON, the `__404.html` error page, and the per-edition `__editions/{slug}.json` metadata files — and uploads them to the project's object store under the `PROJECT` lock. It is triggered by a `publish_edition` completing, by an admin `POST .../dashboard/rebuild`, or by a `dashboard_sync` template fan-out.

A partial unique index, `idx_queue_jobs_dashboard_build_active_uq` on `(org_id, project_id)`, allows at most one active (`queued`/`in_progress`) `dashboard_build` per project. The application-side `DashboardBuildEnqueuer` checks for an active build first and skips redundant enqueues — without this, a keeper-sync backfill that publishes 1,000 editions would cascade 1,000 redundant dashboard rebuilds. When the manual `POST .../dashboard/rebuild` endpoint finds a build already active, it returns **HTTP 409**; the index is the database backstop against a read/create race.

##### Dashboard template sync (`dashboard_sync`)

Triggered by a GitHub webhook when a tracked dashboard-**template** repository is updated. Under the `DASHBOARD_TEMPLATE` lock for the content tuple `(owner, repo, ref, root_path)`, it walks `fetching → writing → fanning_out`: it fetches the template from GitHub (ETag-conditional), upserts the template content and files, and — only if the content changed — **fans out** a `dashboard_build` for every project whose resolved template points at the synced content. It differs from `dashboard_build` in that it operates on the template *source* and rebuilds *many* projects, whereas `dashboard_build` renders *one* project's output. See the {ref}`dashboards` section for the full sync flow.

##### Project GitHub resolution (`project_github_resolve`)

A lightweight, fire-and-forget job enqueued after a project is created or updated with a GitHub binding. It opportunistically resolves the project's GitHub App installation ID and numeric owner/repo IDs. Failures are logged, never retried into an error: the columns stay null and a later App-installation webhook backfills them. It creates no `queue_jobs` row.

##### Health check (`ping`)

A trivial worker that returns `"pong"`, used to verify a pool's workers are alive. No `queue_jobs` row.

(job-keeper-sync)=

#### Keeper-sync pool

The keeper-sync subsystem migrates documentation from the legacy LTD ("Keeper") platform into Docverse (see {ref}`migration`). It serves two purposes with one per-project worker: an operator-triggered **backfill** that copies an LTD instance into Docverse, and a cron-driven **steady-state reconciler** that keeps already-migrated resources fresh while LTD and Docverse run side by side. It runs on its own pool so a large backfill cannot starve live publishing.

##### State and idempotency

A `keeper_sync_state` row records each LTD↔Docverse pairing (one per migrated project, edition, or build): the LTD id/slug, the linked Docverse id, content hashes and ETags, last-synced and last-rebuilt timestamps, and per-tier polling stamps. This row is the idempotency key — a re-sync of unchanged LTD state short-circuits — and the substrate the steady-state tiers poll over. A **tombstone** on the state row (`date_tombstoned` plus a reason: `manual_delete`, `lifecycle_delete`, or `lifecycle_preemptive`) is a permanent veto telling the sync engine "this LTD resource was deleted on the Docverse side; do not re-migrate it." All discovery paths drop tombstoned slugs from their fan-out, and an admin API can list and clear tombstones (clearing also revives the soft-deleted Docverse row).

##### Backfill (`keeper_sync_run_discovery` → `keeper_sync_project`)

An operator starts a backfill by creating a **run** (`POST .../keeper-sync/runs`), which records a `keeper_sync_runs` row and enqueues a `keeper_sync_run_discovery` job. Discovery loads the org's allowlist, fetches LTD's product list, drops out-of-scope and tombstoned slugs, and **fans out** one `keeper_sync_project` child per in-scope project — each attributed to the run via `keeper_sync_run_id`, with the LTD slug in `subject_label`. Each `keeper_sync_project` child syncs one LTD product into Docverse (project → editions → builds), copying build content into the object store and publishing each synced edition through a `publish_edition` job (also attributed to the run). When the last attributed child reaches a terminal state, the run finalizes to `succeeded` or `partial_failure`.

A partial unique index, `idx_queue_jobs_keeper_sync_project_active_uq` on `(org_id, subject_label)`, allows at most one active `keeper_sync_project` per `(org, LTD slug)`, so two concurrent syncs of the same product cannot race on edition creation.

##### Steady-state tiers (`keeper_sync_tier_main` / `_discovery` / `_other`)

Three cron-driven tier reconcilers keep migrated resources fresh without an operator run. They enqueue `keeper_sync_project` children with **no run attribution** (`keeper_sync_run_id` null), and a per-project dormancy gate keeps the long tail of ~1,500 projects from pinning LTD (hot projects poll on the fast cadence; dormant ones fall back to roughly daily, with slug-keyed jitter):

| Tier                          | Cadence  | Targets                                                                |
| ----------------------------- | -------- | --------------------------------------------------------------------- |
| `keeper_sync_tier_main`       | 5 min    | `main` editions whose LTD `date_rebuilt` advanced — keeps the user-visible default edition fresh per the migration SLO |
| `keeper_sync_tier_discovery`  | 30 min   | LTD resources that have no `keeper_sync_state` row yet                 |
| `keeper_sync_tier_other`      | hourly   | non-`main` editions whose state has aged past the refresh threshold   |

##### Reaper (`keeper_sync_reaper`)

A cron backstop (every 30 minutes) that finalizes silently-stuck rows when Arq loses a job entirely (e.g., an OOM-killed worker pod) and no per-job timeout ever fires. In one transaction it fails timed-out run-attributed children (then finalizes their runs), fails timed-out tier-cron children, and fails orphaned tier-cron rows that never got a backend job. See {ref}`reaper-pattern`.

#### Lifecycle pool

Two periodic maintenance subsystems share this pool. Both use the same dispatcher → per-org fan-out, the same per-org mutex, and the same reaper machinery, but own **disjoint** rule sets, so a rule can only ever fire from its own subsystem.

##### Lifecycle evaluation (`lifecycle_eval_dispatcher` → `lifecycle_eval`)

The `lifecycle_eval_dispatcher` runs **hourly**. It records a `lifecycle_eval_runs` row, then fans out one `lifecycle_eval` child per organization that has lifecycle rules configured (skipping orgs with none, so they get no row). Each per-org `lifecycle_eval` child evaluates two rules over the org's projects:

- **Stale drafts** (`DraftInactivityRule`) — draft editions, not lifecycle-exempt, untouched longer than the configured inactivity window.
- **Orphan builds** (`BuildHistoryOrphanRule`) — builds no longer pointed at by any edition and aged out of the recent build-history window.

Matches are **soft-deleted** (the resource's `date_deleted` is set; editions are also unpublished from the CDN, and tombstoned against keeper-sync re-migration). Projects whose edition list shrank get a `dashboard_build` enqueued. The structured log line is the audit trail in this version; persistent delete-reason columns are deferred.

##### Git-ref audit (`git_ref_audit_discovery` → `git_ref_audit`)

The `git_ref_audit_discovery` tick runs **daily at 05:17 UTC** (and is Phalanx-feature-gated — the cron stays registered but no-ops when disabled). It records a `git_ref_audit_runs` row and fans out one `git_ref_audit` child per organization that owns a GitHub-bound project. Each per-org child fetches the live ref set from GitHub for each bound project and evaluates a single rule:

- **Deleted refs** (`RefDeletedRule`) — draft editions tracking a `git_ref` / `alternate_git_ref` whose branch or tag no longer exists on the repository are soft-deleted (same unpublish + tombstone path as `lifecycle_eval`).

This catches ref deletions that a missed GitHub webhook would otherwise leave stranded. A per-project GitHub fetch failure isolates to that project: the child completes `completed_with_errors`, rolling its run to `partial_failure`, without aborting the rest of the org's audit. `git_ref_audit` is a distinct concern from `lifecycle_eval` — it requires GitHub I/O and runs daily — but deliberately reuses the same fan-out, mutex, and reaper patterns rather than competing for worker capacity on the default pool.

##### Per-org mutex

Both subsystems use a single-column partial unique index on `org_id` (`idx_queue_jobs_lifecycle_eval_active_uq` and `idx_queue_jobs_git_ref_audit_active_uq`), allowing at most one active per-org child per kind. Unlike `keeper_sync_project`'s mutex, `subject_label` is **not** part of the identity here: lifecycle-eval and git-ref-audit are per-org by design, with no sub-key under the org, so the org id alone is the mutex. (The row still carries `subject_label = org.slug` for readability.) A slow per-org pass therefore cannot be doubled up by the next tick.

(reaper-pattern)=

#### The reaper pattern

Arq's per-job `timeout` covers a worker that runs too long, but if a worker pod is OOM-killed mid-job — or the producer crashed between committing the `queue_jobs` row and enqueuing the Arq job — no timeout ever fires, and the row is wedged `in_progress` (or `queued` with no `backend_job_id`) forever. For a kind that holds an active mutex, that wedged row also blocks every future enqueue for the same key. **Reapers** are cron jobs that sweep these stuck rows and finalize them.

Each reaper performs two sweeps in one transaction:

- **Silent** — rows `in_progress` past a per-kind staleness threshold are marked `failed` with an error `type` of `SilentWorker`.
- **Orphan** — rows `queued` with `backend_job_id IS NULL`, older than a shared 5-minute window, are marked `failed` with type `OrphanedQueueJob`.

The default-pool kinds get **run-less** reapers (a shared helper, one thin module per kind so each can have its own cron stagger and operator narrative). The fan-out subsystems get **run-aware** reapers that additionally finalize the parent run after failing its stuck children. To keep one horizontally scaled pool from co-firing two reapers against `queue_jobs` at the same instant, the lifecycle-pool reapers are staggered onto disjoint minute slots.

| Reaper                    | Targets kind(s)                       | Silent threshold | Cron cadence (minute)        |
| ------------------------- | ------------------------------------- | ---------------- | ---------------------------- |
| `dashboard_build_reaper`  | `dashboard_build`                     | 30 min           | every 15 min — `{3,18,33,48}` |
| `publish_edition_reaper`  | `publish_edition`                     | 4 h              | every 30 min — `{6,36}`       |
| `build_processing_reaper` | `build_processing`                    | 8 h              | every 30 min — `{12,42}`      |
| `dashboard_sync_reaper`   | `dashboard_sync`                      | 6 h              | every 30 min — `{24,54}`      |
| `lifecycle_reaper`        | `lifecycle_eval` **and** `git_ref_audit` | 6 h           | every 30 min — `{0,30}`       |
| `keeper_sync_reaper`      | `keeper_sync_project` (+ tier-cron)   | 6 h              | every 30 min — `{0,30}`       |

`dashboard_build` gets the tightest cadence because it is the only main-pool kind whose wedge is user-visible (the 409 on `POST .../dashboard/rebuild`); its 30-minute threshold plus 15-minute cron caps worst-case recovery near 45 minutes. `build_processing`'s 8-hour threshold is deliberately generous so a genuinely large multi-hour upload is never falsely reaped. All thresholds are configurable per deployment.

(planned-periodic-jobs)=

#### Planned periodic jobs

Three periodic jobs are designed but **not yet implemented**. They are documented here and referenced elsewhere in this note; when built, each will follow the same Arq-cron pattern as the jobs above.

- **Purgatory cleanup** (`purgatory_cleanup`) — hard-delete object-store content that has sat in purgatory past the org's retention period. (Lifecycle and audit jobs currently soft-delete; reclaiming the underlying storage is this job's responsibility.)
- **Credential re-encryption** (`credential_reencrypt`) — iterate every stored organization credential and re-encrypt it under the current primary Fernet key after a key rotation (see {ref}`periodic-job-scheduling` and the credential-rotation discussion in the organizations section).
- **Inventory census** (`inventory_census`) — a read-only snapshot of resource *levels* (active project / edition / build counts and storage footprint per org and project) that emits the `resource_inventory` metric (see {ref}`metrics-inventory`). Because it only reads, it needs none of the advisory-lock / supersession machinery the mutating jobs use.

(periodic-job-scheduling)=

### Periodic job scheduling

Periodic and orchestration jobs are scheduled with **Arq's built-in cron** (`arq.cron`), declared as `cron_jobs` on the worker settings for the pool that owns them. Each cron tick runs directly in the worker process — the tier reconcilers and the fan-out dispatchers create their run rows and enqueue children inline, with no separate scheduler deployment to operate.

```{code-block} python
:caption: Cron declarations on a worker's settings (illustrative)
class LifecycleEvalWorkerSettings:
    cron_jobs = [
        cron(lifecycle_eval_dispatcher, minute={0}),       # hourly
        cron(git_ref_audit_discovery, hour={5}, minute={17}),  # daily 05:17 UTC
        cron(lifecycle_reaper, minute={0, 30}),            # every 30 min
        # ... staggered run-less reaper backstops ...
    ]
```

The keeper-sync tier cadences are derived from interval constants in the scheduler module and converted to Arq `minute={…}` sets by a shared helper, so the cron schedule and the planner's "next tick" math cannot drift.

#### Schedule table

| Job                          | Pool        | Cadence                       | Purpose                                                       |
| ---------------------------- | ----------- | ----------------------------- | ------------------------------------------------------------ |
| `keeper_sync_tier_main`      | keeper-sync | every 5 min                   | Refresh migrated `main` editions whose LTD build advanced     |
| `keeper_sync_tier_discovery` | keeper-sync | every 30 min                  | Discover LTD resources with no sync-state row yet             |
| `keeper_sync_tier_other`     | keeper-sync | hourly                        | Refresh non-`main` editions aged past the threshold           |
| `keeper_sync_reaper`         | keeper-sync | every 30 min                  | Finalize stuck keeper-sync jobs and their runs                |
| `lifecycle_eval_dispatcher`  | lifecycle   | hourly                        | Fan out per-org lifecycle-eval children                       |
| `git_ref_audit_discovery`    | lifecycle   | daily, 05:17 UTC              | Fan out per-org git-ref-audit children (feature-gated)        |
| `lifecycle_reaper`           | lifecycle   | every 30 min                  | Finalize stuck `lifecycle_eval` + `git_ref_audit` jobs/runs   |
| reaper backstops             | lifecycle   | 15–30 min, staggered          | Finalize stuck default-pool jobs (see {ref}`reaper-pattern`)  |

Cadences, the daily audit window, and feature gates are configurable per deployment via Phalanx Helm values, so operators can retune frequencies or disable a job without code changes. The **planned** periodic jobs ({ref}`planned-periodic-jobs`) will join this table when implemented — e.g. `purgatory_cleanup` daily, `credential_reencrypt` weekly, `inventory_census` daily.

> **Design note.** An earlier revision of this section proposed scheduling periodic jobs with **Kubernetes CronJobs** that shell out to a `docverse-admin enqueue <type>` CLI, to decouple scheduling from the queue backend. The implementation uses Arq cron instead. With a large and growing set of distinct periodic and orchestration jobs, declaring each as a Kubernetes CronJob would push substantial complexity into the Phalanx Helm charts, which are managed separately from the application — every new job, cadence change, or feature gate would become a chart edit. Keeping the schedule in code alongside the workers makes adding and tuning jobs a single-repository change, and the per-pool isolation already prevents cron work from interfering with latency-sensitive jobs. The trade-off is that the cron schedule lives with the queue backend rather than in Kubernetes; a backend swap would re-declare the crons against the new backend's scheduling primitive.

### Failure and retry

Retry behavior is configured per pool and per job type:

- **Default-pool jobs** (`build_processing`, `publish_edition`, `dashboard_build`, `dashboard_sync`) use Arq's default retry-with-backoff. They are idempotent on retry — inventory upserts, deterministic tracking, and diff-based publishes mean a re-run of a completed step is effectively a no-op.
- **Fan-out workers on the dedicated pools** (`keeper_sync_run_discovery`, `keeper_sync_project`, `lifecycle_eval_dispatcher`, `lifecycle_eval`, `git_ref_audit_discovery`, `git_ref_audit`) run with **`max_tries=1`** and an explicit per-job **timeout**, rather than Arq's 5-attempt default. A failure must surface *promptly* so the worker's `except` block can route to `queue_job_store.fail()` and the parent run can finalize via its finaliser; a silent multi-attempt retry would only delay finalization and bury the error. The per-job timeout is the first backstop against a runaway job; the cron reaper ({ref}`reaper-pattern`) is the second, for the case where Arq loses the job entirely and no timeout fires.

Within build processing, edition publishing is fanned out, so a single edition's publish failure is isolated to its own `publish_edition` child — the build job itself completes, and the failed child is retried or addressed independently. Edition-tracking failure inside `build_processing` is non-fatal: the build is marked `completed_with_errors` and no publish children are spawned.

### Job retention

Completed and failed `queue_jobs` records are retained for a configurable period (default: 7 days) before cleanup. The queue API returns 404 for expired jobs. Reaping object-store content left behind by soft-deleted resources is the responsibility of the planned `purgatory_cleanup` job ({ref}`planned-periodic-jobs`).
