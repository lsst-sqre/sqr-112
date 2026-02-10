(queue)=

## Task queue design

### Design philosophy

Docverse uses [oban-py](https://github.com/oban-bg/oban-py) strictly as a **distributed job queue** — it handles scheduling, delivery, retries, and persistence. All orchestration and parallelism within a job is handled by Docverse's service layer using standard Python asyncio. This minimizes coupling to oban-py (no dependency on oban-pro's workflow system) and keeps the business logic testable with plain async functions.

Each user-facing operation that triggers background work results in a **single oban job**. The job's `perform` function calls through the service layer, which coordinates the steps internally. Where steps are independent, the service layer uses `asyncio.gather()` to parallelize them.

### QueueJob table

Docverse maintains its own `QueueJob` table in Postgres as the single source of truth for job state and progress. This table serves the user-facing queue API, operator dashboards, and internal coordination (e.g., detecting conflicting concurrent edition updates). oban-py's internal job table is not queried directly for status — Docverse treats oban as a delivery mechanism only.

| Column           | Type                    | Description                                                                                                    |
| ---------------- | ----------------------- | -------------------------------------------------------------------------------------------------------------- |
| `id`             | int                     | Internal PK                                                                                                    |
| `public_id`      | int                     | Crockford Base32 serialized in API                                                                             |
| `oban_job_id`    | int                     | Reference to oban's internal job ID                                                                            |
| `kind`           | enum                    | `build_processing`, `edition_update`, `dashboard_sync`, `lifecycle_eval`, `git_ref_audit`, `purgatory_cleanup` |
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

During the editions phase, multiple edition update coroutines run concurrently via `asyncio.gather()`. Each coroutine updates the `progress` JSONB when it completes, using Postgres `jsonb_set` to atomically move its edition slug between the `editions_in_progress` and `editions_completed` (or `editions_failed`) arrays:

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

Postgres serializes the row locks, but since these are sub-millisecond metadata writes against a single row, contention is negligible compared to the actual edition update work (KV writes, cache purges, or object copies).

The service layer wraps each edition update coroutine:

```python
async def _update_single_edition(self, edition, build, job_id):
    try:
        await self._edition_service.update(edition, build)
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
  "editions_completed": ["__main", "v2.x"],
  "editions_failed": [
    { "slug": "DM-12345", "error": "R2 timeout after 3 retries" }
  ],
  "editions_in_progress": []
}
```

For other phases, progress can carry simpler metadata (e.g., `{"message": "Cataloging 1,247 objects"}` during inventory).

### Operator queries

The `QueueJob` table provides a single place for operators to understand system state across all workers:

- **Backlog depth**: `SELECT count(*), kind FROM queue_job WHERE status = 'queued' GROUP BY kind`
- **Active work**: `SELECT * FROM queue_job WHERE status = 'in_progress'` — shows what every worker is doing, which phase each job is in, and per-edition progress
- **Edition conflict detection**: `SELECT * FROM queue_job WHERE status = 'in_progress' AND project_id = :pid AND phase = 'editions'` — inspect the `progress` JSONB to check if a specific edition is currently being updated before starting another update
- **Error rates**: `SELECT count(*) FROM queue_job WHERE status IN ('failed', 'completed_with_errors') AND date_completed > now() - interval '1 hour'`
- **Per-org throughput**: `SELECT org_id, count(*) FROM queue_job WHERE status = 'completed' AND date_completed > now() - interval '1 hour' GROUP BY org_id`
- **Slow jobs**: `SELECT * FROM queue_job WHERE status = 'in_progress' AND date_started < now() - interval '10 minutes'`

### Job types

#### Build processing (`build_processing`)

Triggered when a client signals upload complete (`PATCH .../builds/:build` with `status: uploaded`). This is the primary job type.

The service layer executes the following steps inside the single oban job:

1. **Inventory** (sequential) — catalog the build's objects from the object store into the `BuildObject` table in Postgres (key, content hash, content type, size). This is a listing + metadata operation against the object store.

2. **Evaluate tracking rules** (sequential) — determine which editions should update based on the build's git ref, the project's edition tracking modes, and the org's rewrite rules. Auto-create new editions if needed (e.g., new semver major stream, new git ref). Returns a list of affected editions.

3. **Update editions** (parallel via `asyncio.gather()`) — for each affected edition, update the edition to point to the new build. In pointer mode this writes a new KV mapping and purges the CDN cache; in copy mode this performs the ordered diff-copy-purge sequence. Each edition update also logs the transition to the `EditionBuildHistory` table. If one edition update fails, the others continue to completion; failures are collected and reported via the `QueueJob` progress JSONB.

4. **Render project dashboard** (sequential, runs once after all edition updates complete) — re-render the project's dashboard and 404 pages using the current edition metadata from the database and the resolved template. A single build may update multiple editions, but the dashboard reflects the project's full edition list and only needs to be rendered once.

5. **Update job status** (sequential) — mark the `QueueJob` as `completed`, `completed_with_errors` (if some editions failed), or `failed`. Also update the build's status accordingly.

#### Edition reassignment (`edition_update`)

Triggered when an admin PATCHes an edition with a new `build` field (manual reassignment or rollback). Simpler than build processing — a single oban job that:

1. Updates the edition to point to the specified build (pointer mode KV write or copy mode diff-copy).
2. Logs the transition to `EditionBuildHistory`.
3. Renders the project dashboard.

#### Dashboard template sync (`dashboard_sync`)

Triggered by a GitHub webhook when a tracked dashboard template repository is updated. A single oban job that syncs the template files from GitHub to the object store, then re-renders dashboards for all affected projects (all projects in the org for an org-level template, or a single project for a project-level override), using `asyncio.gather()` to parallelize across projects. See the {ref}`dashboards` section for the full sync flow.

#### Lifecycle evaluation (`lifecycle_eval`)

Scheduled periodically via oban cron. A single oban job that scans all orgs and projects for editions and builds matching lifecycle rules (stale drafts, orphan builds). Soft-deletes matching resources and moves object store content to purgatory. Uses `asyncio.gather()` to parallelize across orgs.

#### Git ref audit (`git_ref_audit`)

Scheduled periodically via oban cron. A single oban job that verifies git refs tracked by editions still exist on their GitHub repositories. Flags or soft-deletes editions whose refs have been deleted (if the `ref_deleted` lifecycle rule is enabled). Catches cases where GitHub webhook delivery for ref deletion events was missed.

#### Purgatory cleanup (`purgatory_cleanup`)

Scheduled periodically via oban cron. A single oban job that hard-deletes object store objects that have been in purgatory longer than the org's configured retention period. Simple listing + batch delete per org.

#### Credential rewrap (`credential_rewrap`)

Scheduled periodically via oban cron. A single oban job that iterates over all `organization_credentials` rows, checks whether the `vault_ciphertext` token uses the latest key version (by inspecting the `vault:vN:...` prefix against the key's current version from Vault), and rewraps any stale entries. This ensures that after a key rotation, all stored credentials are migrated to the new key version without ever exposing plaintext. Parallelized across orgs via `asyncio.gather()`.

### Failure and retry

oban-py handles job-level retries. The retry policy varies by job type:

- **build_processing** and **edition_update**: retry with backoff, up to 3 attempts. Jobs are idempotent at each step — inventory upserts, tracking evaluation is deterministic, edition updates use diffs (already-updated editions show no changes on re-run). On retry, the job re-runs from the beginning but completed steps are effectively no-ops. The `QueueJob` progress is reset at the start of each attempt.
- **Periodic jobs** (lifecycle_eval, git_ref_audit, purgatory_cleanup): retry once, then wait for the next scheduled run. These are self-correcting — anything missed on one run will be caught on the next.

Within a build processing job, individual edition update failures (in the `asyncio.gather()`) do not fail the entire job. The service layer uses `return_exceptions=True`, collects results, and marks the job as `completed_with_errors` if some editions failed while others succeeded. Failed editions are recorded in the `progress` JSONB with error messages for diagnosis. A subsequent retry or manual edition PATCH can address individual failures.

### Job retention

Completed and failed `QueueJob` records are retained for a configurable period (default: 7 days) before being cleaned up by the purgatory cleanup job. The queue API returns 404 for expired jobs.
