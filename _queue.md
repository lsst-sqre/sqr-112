(queue)=

## Task queue design

### Design philosophy

Docverse interacts with the job queue through a **backend-agnostic abstraction layer** (see {ref}`queue-backend-protocol`). The initial implementation uses [Arq](https://arq-docs.helpmanual.io/) via [Safir's ArqQueue](https://safir.lsst.io/) with Redis as the message transport. The queue backend handles delivery, retries, and worker dispatch. All orchestration and parallelism within a job is handled by Docverse's service layer using standard Python asyncio. This minimizes coupling to any specific queue technology and keeps the business logic testable with plain async functions.

Each user-facing operation that triggers background work results in a **single background job**. The job's worker function calls through the service layer, which coordinates the steps internally. Where steps are independent, the service layer uses `asyncio.gather()` to parallelize them.

### QueueJob table

Docverse maintains its own `QueueJob` table in Postgres as the single source of truth for job state and progress. This table serves the user-facing queue API, operator dashboards, and internal coordination (e.g., detecting conflicting concurrent edition updates). The queue backend's internal state is not queried directly for status — Docverse treats the backend as a delivery mechanism only. See {ref}`table-queue-job` in the database schema section for the column reference within the full schema.

| Column           | Type                    | Description                                                                                                    |
| ---------------- | ----------------------- | -------------------------------------------------------------------------------------------------------------- |
| `id`             | int                     | Internal PK                                                                                                    |
| `public_id`      | int                     | Crockford Base32 serialized in API                                                                             |
| `backend_job_id` | str (nullable)          | Reference to the queue backend's job ID (e.g., Arq UUID)                                                       |
| `kind`           | enum                    | `build_processing`, `edition_update`, `dashboard_sync`, `lifecycle_eval`, `git_ref_audit`, `purgatory_cleanup`, `credential_reencrypt` |
| `status`         | enum                    | `queued`, `in_progress`, `completed`, `completed_with_errors`, `failed`, `cancelled`                           |
| `phase`          | str (nullable)          | Current phase: `inventory`, `tracking`, `editions`, `dashboard`                                                |
| `org_id`         | FK → Organization       | Scoped to org (for operator filtering)                                                                         |
| `project_id`     | FK → Project (nullable) | Set for build/edition jobs                                                                                     |
| `build_id`       | FK → Build (nullable)   | Set for build processing jobs                                                                                  |
| `progress`       | JSONB (nullable)        | Structured progress data, phase-specific                                                                       |
| `errors`         | JSONB (nullable)        | Collected error details                                                                                        |
| `date_created`   | datetime                | When enqueued                                                                                                  |
| `date_started`   | datetime (nullable)     | When a worker picked it up                                                                                     |
| `date_completed` | datetime (nullable)     | When finished                                                                                                  |

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

**`ArqQueueBackend`** wraps Safir's `ArqQueue` for production use. Arq uses UUID strings as job IDs, which are stored in the `backend_job_id` column of the `QueueJob` table. The worker functions are standard async functions that receive the job payload and call through the service layer.

**`MockQueueBackend`** wraps Safir's `MockArqQueue` for testing. Jobs are executed in-process, making tests deterministic without requiring a running Redis instance.

Both implementations are constructed by the factory and injected into services, consistent with Docverse's dependency injection pattern.

#### Infrastructure

Arq requires a **Redis** instance as its message broker. In Phalanx deployments, Redis is a standard in-cluster service. The `QueueJob` Postgres table remains the authoritative state store — Redis holds only transient message data. If Redis state is lost, in-flight jobs can be re-enqueued from `QueueJob` records with `status = 'queued'`.

### Progress tracking

The service layer writes progress to the `QueueJob` table at each phase transition and within phases where granular tracking is useful. These are lightweight single-row UPDATEs.

#### Phase transitions

At each major phase boundary, the service updates the `phase` column and resets/initializes the `progress` JSONB:

```python
await queue_job_store.update_phase(job_id, "inventory")
await inventory_service.catalog_build(build)

await queue_job_store.update_phase(job_id, "tracking")
affected_editions = await tracking_service.evaluate(build)

await queue_job_store.start_editions_phase(job_id, affected_editions)
results = await asyncio.gather(
    *[self._update_single_edition(e, build, job_id)
      for e in affected_editions],
    return_exceptions=True,
)

await queue_job_store.update_phase(job_id, "dashboard")
await dashboard_service.render(project)

await queue_job_store.complete(job_id)
```

#### Live edition progress via conditional JSONB merge

During the editions phase, multiple edition update coroutines run concurrently via `asyncio.gather()`. Each coroutine updates the `progress` JSONB when it completes, using Postgres `jsonb_set` to atomically move its edition slug between the `editions_in_progress` and `editions_completed` (or `editions_skipped` or `editions_failed`) arrays:

```sql
-- Mark edition completed
UPDATE queue_job
SET progress = jsonb_set(
    jsonb_set(
        progress,
        '{editions_completed}',
        (progress->'editions_completed') || to_jsonb(:edition_slug::text)
    ),
    '{editions_in_progress}',
    (progress->'editions_in_progress') - :edition_slug
)
WHERE id = :job_id
```

For failures, the slug is moved to `editions_failed` as a structured object with error context:

```sql
-- Mark edition failed
UPDATE queue_job
SET progress = jsonb_set(
    jsonb_set(
        progress,
        '{editions_failed}',
        (progress->'editions_failed') || :failed_entry::jsonb
    ),
    '{editions_in_progress}',
    (progress->'editions_in_progress') - :edition_slug
)
WHERE id = :job_id
```

Where `failed_entry` is `{"slug": "DM-12345", "error": "R2 timeout after 3 retries"}`.

For skipped editions (superseded by a newer build; see {ref}`cross-job-serialization`), the slug is moved to `editions_skipped` as a structured object with the reason:

```sql
-- Mark edition skipped
UPDATE queue_job
SET progress = jsonb_set(
    jsonb_set(
        progress,
        '{editions_skipped}',
        (progress->'editions_skipped') || :skipped_entry::jsonb
    ),
    '{editions_in_progress}',
    (progress->'editions_in_progress') - :edition_slug
)
WHERE id = :job_id
```

Where `skipped_entry` is `{"slug": "v2.x", "reason": "superseded by build 01HQ-3KBR-T5GN-8W"}`.

Postgres serializes the row locks, but since these are sub-millisecond metadata writes against a single row, contention is negligible compared to the actual edition update work (KV writes, cache purges, or object copies).

The service layer wraps each edition update coroutine:

```python
async def _update_single_edition(self, edition, build, job_id):
    try:
        skipped = await self._edition_service.update(edition, build)
        if skipped:
            await self._queue_store.mark_edition_skipped(
                job_id, edition.slug, reason="superseded"
            )
        else:
            await self._queue_store.mark_edition_completed(
                job_id, edition.slug
            )
    except Exception as e:
        await self._queue_store.mark_edition_failed(
            job_id, edition.slug, str(e)
        )
        raise
```

#### Progress JSONB structure

The `progress` JSONB is phase-specific. During the editions phase:

```json
{
  "editions_total": 3,
  "editions_completed": ["__main"],
  "editions_skipped": [
    { "slug": "v2.x", "reason": "superseded by build 01HQ-3KBR-T5GN-8W" }
  ],
  "editions_failed": [
    { "slug": "DM-12345", "error": "R2 timeout after 3 retries" }
  ],
  "editions_in_progress": []
}
```

For other phases, progress can carry simpler metadata (e.g., `{"message": "Cataloging 1,247 objects"}` during inventory).

(cross-job-serialization)=

### Cross-job serialization

Several background jobs can race on the same project's resources. Two rapid build uploads from the same branch can produce two `build_processing` jobs that both try to update the same edition concurrently. A `build_processing` job and a `dashboard_sync` job can both try to write the same project's dashboard files at the same time. An `edition_update` job and a `build_processing` job can both write the same edition's metadata JSON. Since `asyncio.gather()` parallelizes edition updates *within* a job, and multiple workers can process different jobs simultaneously, these concurrent mutations can lead to interleaved KV writes, partial cache purges, torn dashboard HTML, or inconsistent metadata JSON.

Docverse prevents this with **Postgres advisory locks** at two granularities — per-edition and per-project — combined with a **stale build guard** for edition updates.

#### Lock namespacing

Advisory locks use the two-argument form `pg_advisory_lock(classid, objid)` to namespace by resource type, avoiding key collisions between edition and project PKs (which come from different sequences):

- `pg_advisory_lock(1, edition.id)` — edition-level lock, serializes edition content updates and per-edition metadata JSON writes.
- `pg_advisory_lock(2, project.id)` — project-level lock, serializes project-wide dashboard renders (`__dashboard.html`, `__404.html`, `__switcher.json`).

Both services acquire locks through a shared `advisory_lock` async context manager that wraps the acquire/release pair, making the lock scope visually explicit and eliminating repeated `try`/`finally` boilerplate:

```python
@asynccontextmanager
async def advisory_lock(session, classid, objid):
    """Acquire a Postgres advisory lock for the duration of the block."""
    await session.execute(
        text("SELECT pg_advisory_lock(:classid, :objid)"),
        {"classid": classid, "objid": objid},
    )
    try:
        yield
    finally:
        await session.execute(
            text("SELECT pg_advisory_unlock(:classid, :objid)"),
            {"classid": classid, "objid": objid},
        )
```

#### Advisory lock acquisition

Before performing any mutation, `EditionService.update()` uses the `advisory_lock` context manager to hold an advisory lock keyed on the edition's primary key for the duration of the update:

```python
async def update(self, edition, build) -> bool:
    """Update edition to point to build. Returns True if skipped."""
    async with advisory_lock(self._session, 1, edition.id):
        current_build = await self._get_current_build(edition)
        if current_build and current_build.date_created > build.date_created:
            return True  # Skipped — edition already has a newer build

        # ... perform KV write / copy-mode update ...
        # ... log to EditionBuildHistory ...
        # ... write __editions/{slug}.json metadata ...
        return False
```

The underlying `pg_advisory_lock()` call blocks (rather than failing) if another session holds the lock for the same key. This means a competing job simply waits its turn — no job is rejected or fails due to contention.

#### Stale build guard

After acquiring the lock, the method compares the candidate build's `date_created` against the edition's current build. If the edition already points to a newer build (because a more recent job acquired the lock first), the update is skipped. The caller logs the skip in the job's `progress` JSONB via `mark_edition_skipped`, and the edition slug appears in the `editions_skipped` array rather than `editions_completed`.

This guarantees the edition never regresses to an older build, regardless of the order in which competing jobs acquire the lock.

#### Project-level lock for dashboard renders

`DashboardService.render(project)` acquires a project-level advisory lock before writing the project-wide dashboard files. After releasing the project lock, it acquires each edition's lock in turn to write per-edition metadata JSON, serializing against any concurrent `EditionService.update()` that writes the same file.

```python
async def render(self, project):
    """Render all dashboard outputs for a project."""
    # Project-wide files under project lock
    async with advisory_lock(self._session, 2, project.id):
        await self._write_dashboard_html(project)
        await self._write_404_html(project)
        await self._write_switcher_json(project)

    # Per-edition metadata under individual edition locks
    for edition in await self._get_editions(project):
        async with advisory_lock(self._session, 1, edition.id):
            await self._write_edition_metadata_json(edition)
```

No stale guard is needed for dashboard renders. The dashboard output is deterministic from the current database state, so the last render to complete always produces the correct output. The per-edition metadata writes can be parallelized across editions (different lock keys), but each individual write serializes against any concurrent `EditionService.update()` for the same edition.

#### Why this works

- **No concurrent mutation**: The edition-level advisory lock serializes all updates to a given edition, whether from `build_processing` or `edition_update` jobs. The project-level lock serializes all dashboard renders for a project, whether from build processing, edition updates, template syncs, or manual re-renders.
- **No failures**: `pg_advisory_lock()` blocks until the lock is available — the job waits rather than failing.
- **Correct final state**: If Build B (newer) is processed before Build A (older) due to lock acquisition order, Build A's update is skipped by the stale guard. The edition always reflects the most recent build. Dashboard renders are deterministic from database state, so the last render to complete is always correct.
- **Compatible with `asyncio.gather()`**: Each edition's lock is independent, so parallel updates of *different* editions within the same job proceed without contention. Only updates to the *same* edition across jobs serialize. Similarly, `dashboard_sync` jobs that re-render multiple projects in parallel acquire independent project-level locks.
- **Covers all code paths**: Placing the edition lock inside `EditionService.update()` covers both the `build_processing` parallel edition phase and the `edition_update` manual reassignment path. Placing the project lock inside `DashboardService.render()` covers all dashboard render triggers. Per-edition metadata JSON is protected in both locations — inside `EditionService.update()` and during `DashboardService.render()`'s per-edition loop.

#### Connection impact

The advisory lock holds a database session open for the duration of the locked operation. For edition updates in pointer mode (~2 seconds for KV write + cache purge) this is negligible. For copy mode (longer due to object copies), the session is held longer but this is acceptable given expected concurrency levels — at most a few concurrent builds per project. The project-level dashboard lock is held only for the duration of writing the three project-wide files (HTML + JSON), which is sub-second — significantly shorter than edition content updates.

### Operator queries

The `QueueJob` table provides a single place for operators to understand system state across all workers:

- **Backlog depth**: `SELECT count(*), kind FROM queue_job WHERE status = 'queued' GROUP BY kind`
- **Active work**: `SELECT * FROM queue_job WHERE status = 'in_progress'` — shows what every worker is doing, which phase each job is in, and per-edition progress
- **Edition update activity**: `SELECT * FROM queue_job WHERE status = 'in_progress' AND project_id = :pid AND phase = 'editions'` — shows concurrent edition work for a project. Advisory locks (see {ref}`cross-job-serialization`) handle serialization automatically; this query is for observability
- **Error rates**: `SELECT count(*) FROM queue_job WHERE status IN ('failed', 'completed_with_errors') AND date_completed > now() - interval '1 hour'`
- **Per-org throughput**: `SELECT org_id, count(*) FROM queue_job WHERE status = 'completed' AND date_completed > now() - interval '1 hour' GROUP BY org_id`
- **Slow jobs**: `SELECT * FROM queue_job WHERE status = 'in_progress' AND date_started < now() - interval '10 minutes'`

### Job types

#### Build processing (`build_processing`)

Triggered when a client signals upload complete (`PATCH .../builds/:build` with `status: uploaded`). This is the primary job type.

The service layer executes the following steps inside the single background job:

1. **Inventory** (sequential) — catalog the build's objects from the object store into the `BuildObject` table in Postgres (key, content hash, content type, size). This is a listing + metadata operation against the object store.

2. **Evaluate tracking rules** (sequential) — determine which editions should update based on the build's git ref, the project's edition tracking modes, and the org's rewrite rules. Auto-create new editions if needed (e.g., new semver major stream, new git ref). Returns a list of affected editions.

3. **Update editions** (parallel via `asyncio.gather()`) — for each affected edition, update the edition to point to the new build. In pointer mode this writes a new KV mapping and purges the CDN cache; in copy mode this performs the ordered diff-copy-purge sequence. Each edition update also logs the transition to the `EditionBuildHistory` table. If one edition update fails, the others continue to completion; failures are collected and reported via the `QueueJob` progress JSONB.

4. **Render project dashboard** (sequential, runs once after all edition updates complete) — re-render the project's dashboard and 404 pages using the current edition metadata from the database and the resolved template. A single build may update multiple editions, but the dashboard reflects the project's full edition list and only needs to be rendered once.

5. **Update job status** (sequential) — mark the `QueueJob` as `completed`, `completed_with_errors` (if some editions failed), or `failed`. Also update the build's status accordingly.

#### Edition reassignment (`edition_update`)

Triggered when an admin PATCHes an edition with a new `build` field (manual reassignment or rollback). Simpler than build processing — a single background job that:

1. Updates the edition to point to the specified build (pointer mode KV write or copy mode diff-copy).
2. Logs the transition to `EditionBuildHistory`.
3. Renders the project dashboard.

#### Dashboard template sync (`dashboard_sync`)

Triggered by a GitHub webhook when a tracked dashboard template repository is updated. A single background job that syncs the template files from GitHub to the object store, then re-renders dashboards for all affected projects (all projects in the org for an org-level template, or a single project for a project-level override), using `asyncio.gather()` to parallelize across projects. See the {ref}`dashboards` section for the full sync flow.

#### Lifecycle evaluation (`lifecycle_eval`)

Scheduled periodically (see {ref}`periodic-job-scheduling`). A single background job that scans all orgs and projects for editions and builds matching lifecycle rules (stale drafts, orphan builds). Soft-deletes matching resources and moves object store content to purgatory. Uses `asyncio.gather()` to parallelize across orgs.

#### Git ref audit (`git_ref_audit`)

Scheduled periodically (see {ref}`periodic-job-scheduling`). A single background job that verifies git refs tracked by editions still exist on their GitHub repositories. Flags or soft-deletes editions whose refs have been deleted (if the `ref_deleted` lifecycle rule is enabled). Catches cases where GitHub webhook delivery for ref deletion events was missed.

#### Purgatory cleanup (`purgatory_cleanup`)

Scheduled periodically (see {ref}`periodic-job-scheduling`). A single background job that hard-deletes object store objects that have been in purgatory longer than the org's configured retention period. Simple listing + batch delete per org.

#### Credential re-encryption (`credential_reencrypt`)

Scheduled periodically (see {ref}`periodic-job-scheduling`). A single background job that iterates over all `organization_credentials` rows and calls `CredentialEncryptor.rotate()` on each `encrypted_credential` value. This re-encrypts every token under the current primary Fernet key. Unlike Vault's `vault:vN:` prefix, Fernet tokens don't indicate which key encrypted them, so the job processes all rows unconditionally — `MultiFernet.rotate()` is idempotent. This ensures that after a key rotation, all stored credentials are migrated to the new key without ever exposing plaintext. Parallelized across orgs via `asyncio.gather()`.

(periodic-job-scheduling)=

### Periodic job scheduling

Periodic jobs are scheduled using **Kubernetes CronJobs** rather than the queue backend's built-in scheduling features (e.g., Arq's cron). This keeps scheduling decoupled from the queue backend, enabling backend swaps without changing scheduling infrastructure. Kubernetes CronJobs are well-understood, observable, and already used throughout Phalanx.

Each periodic job type gets a CronJob that runs a thin CLI command to create a `QueueJob` record and enqueue it via the queue backend:

```{code-block} yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: docverse-lifecycle-eval
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: docverse-enqueue
              image: ghcr.io/lsst-sqre/docverse:latest
              command: ["docverse", "enqueue", "lifecycle_eval"]
          restartPolicy: OnFailure
```

The `docverse enqueue` CLI command connects to the database and Redis, creates a `QueueJob` record with `status: queued`, enqueues the job via the queue backend, and exits. The actual work is performed by the Docverse worker process.

#### Schedule table

| Job type           | Default schedule       | Description                                  |
| ------------------ | ---------------------- | -------------------------------------------- |
| `lifecycle_eval`   | Daily at 03:00 UTC     | Evaluate edition and build lifecycle rules    |
| `git_ref_audit`    | Daily at 04:00 UTC     | Verify git refs tracked by editions           |
| `purgatory_cleanup`| Daily at 05:00 UTC     | Hard-delete expired purgatory objects          |
| `credential_reencrypt`| Weekly (Sunday 02:00)  | Re-encrypt credentials under current primary Fernet key |

Schedules are configurable per-deployment via Phalanx Helm values. Operators can adjust frequencies, add maintenance windows, or disable specific jobs without code changes.

### Failure and retry

The queue backend handles job-level retries. With Arq, retry behavior is configured per job type via the worker's job definitions. The retry policy varies by job type:

- **build_processing** and **edition_update**: retry with backoff, up to 3 attempts. Jobs are idempotent at each step — inventory upserts, tracking evaluation is deterministic, edition updates use diffs (already-updated editions show no changes on re-run). On retry, the job re-runs from the beginning but completed steps are effectively no-ops. The `QueueJob` progress is reset at the start of each attempt.
- **Periodic jobs** (lifecycle_eval, git_ref_audit, purgatory_cleanup): retry once, then wait for the next scheduled run. These are self-correcting — anything missed on one run will be caught on the next.

Within a build processing job, individual edition update failures (in the `asyncio.gather()`) do not fail the entire job. The service layer uses `return_exceptions=True`, collects results, and marks the job as `completed_with_errors` if some editions failed while others succeeded. Failed editions are recorded in the `progress` JSONB with error messages for diagnosis. A subsequent retry or manual edition PATCH can address individual failures.

### Job retention

Completed and failed `QueueJob` records are retained for a configurable period (default: 7 days) before being cleaned up by the purgatory cleanup job. The queue API returns 404 for expired jobs.
