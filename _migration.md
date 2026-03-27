(migration)=

## Migration from LSST the Docs

The Rubin Observatory LSST the Docs (LTD) deployment at `lsst.io` serves ~300 documentation projects for the Rubin Observatory software stack and technical notes.
The current deployment runs LTD Keeper 1.23.0 with content stored in AWS S3, served through Fastly CDN, and uploaded via the `lsst-sqre/ltd-upload` GitHub Action and reusable workflows.
The migration moves this deployment to Docverse, targeting Cloudflare R2 for object storage and Cloudflare Workers for the CDN edge.

The migration involves three concerns: **data migration** (moving object store content and database records), **client migration** (updating CI workflows that upload documentation), and a **phased rollout** that minimizes disruption.

### Data migration

The data migration moves documentation content from the LTD object store layout to the Docverse layout and seeds the Docverse database with project, edition, and build records derived from the LTD Keeper database.

**Scope**: Only builds that are currently referenced by active (non-deleted) editions are migrated.
Historical builds that are not pointed to by any edition are discarded.
This significantly reduces the data volume — most projects accumulate hundreds of builds over time, but only a handful of editions (and therefore builds) are active at any given moment.

#### LTD vs. Docverse object store layout

The LTD and Docverse object store layouts differ in both structure and semantics:

| Aspect | LTD layout | Docverse layout |
|---|---|---|
| Build storage | `{product}/builds/{build_id}/{file}` | `{project}/__builds/{build_id}/{file}` |
| Edition content | `{product}/editions/{slug}/{file}` (physical copy of build files) | No edition file copies — editions are pointers resolved at the CDN edge via KV lookup |
| Edition metadata | None in object store (stored in LTD Keeper database only) | `{project}/__editions/{slug}.json` (per-edition metadata for client-side JavaScript) |
| Staging | N/A (files uploaded individually via presigned URLs) | `{project}/__staging/{build_id}.tar.gz` (tarball staging area) |
| Dashboard | Static HTML at domain root | `{project}/__dashboards/{slug}.html` (template-rendered per edition) |

The key architectural difference is that LTD physically copies build files into edition paths on every edition update (the S3 copy-on-publish bottleneck described in {ref}`documentation-hosting`), while Docverse stores builds once and resolves editions to builds via edge KV lookups.
This means the migration only needs to copy files for the builds themselves — edition content does not need to be duplicated.

#### Migration tool design

The migration is implemented as a `docverse migrate` CLI command in the Docverse client package.
The tool reads from the LTD Keeper database and source object store, and writes to the Docverse API and target object store.

```{mermaid}
sequenceDiagram
    participant CLI as docverse migrate
    participant LTD_DB as LTD Keeper DB
    participant S3_SRC as Source S3 Bucket
    participant API as Docverse API
    participant Store as Target Object Store
    participant KV as CDN Edge KV

    CLI->>LTD_DB: Query products, editions, builds
    LTD_DB-->>CLI: Product→Edition→Build mappings

    loop For each product
        CLI->>API: Create project (with org mapping)

        loop For each unique build referenced by active editions
            CLI->>S3_SRC: List objects at {product}/builds/{build_id}/
            S3_SRC-->>CLI: Object keys + metadata

            loop For each object in build
                CLI->>Store: Copy to {project}/__builds/{build_id}/{file}
            end

            CLI->>API: Register build (metadata, object inventory)
        end

        loop For each active edition
            CLI->>API: Create edition (slug, tracking mode, kind)
            CLI->>API: Set edition → build pointer
            API->>KV: Write edition→build mapping
        end
    end

    CLI->>API: Trigger dashboard renders
```

The migration tool proceeds in these steps:

1. **Query LTD Keeper database** for all products and their active edition→build mappings. For each product, determine which builds are referenced by at least one non-deleted edition.

2. **Create Docverse projects** via the API. Map LTD product slugs to Docverse project slugs (typically identical). Associate each project with the appropriate Docverse organization.

3. **Copy build objects** for each unique build referenced by an active edition. Objects are copied from `{product}/builds/{build_id}/` in the source S3 bucket to `{project}/__builds/{build_id}/` in the target object store. The tool populates the `BuildObject` inventory table during this step by recording each object's key, content hash, content type, and size.

4. **Create edition records** in Docverse with mapped tracking modes (see below) and appropriate edition kinds. Create `EditionBuildHistory` entries linking each edition to its current build.

5. **Seed CDN edge data store** by writing edition→build mappings to the KV store, enabling the CDN to resolve edition URLs to the correct build paths immediately.

6. **Render dashboards and metadata JSON** by triggering dashboard render jobs for each migrated project.

#### Tracking mode mapping

LTD Keeper tracking modes map directly to their Docverse equivalents.
The names have changed slightly (e.g., `git_refs` → `git_ref`), but the semantics are preserved:

| LTD Keeper mode | Docverse mode | Notes |
|---|---|---|
| `git_refs` | `git_ref` | Singular form; same behavior |
| `lsst_doc` | `lsst_doc` | Unchanged |
| `eups_major_release` | `eups_major_release` | Unchanged |
| `eups_weekly_release` | `eups_weekly_release` | Unchanged |
| `eups_daily_release` | `eups_daily_release` | Unchanged |

Edition kinds are inferred from the tracking mode and edition slug:
- Editions with slug `__main` (or equivalent main branch tracking) → kind `main`
- Editions with `lsst_doc` or `eups_*` tracking → kind `release`
- All other editions → kind `draft`

#### Rubin-specific notes

The Rubin migration involves a cross-cloud transfer from AWS S3 to Cloudflare R2.
R2 provides an S3-compatible API, so the migration tool can use standard S3 client libraries (boto3/aioboto3) for both source reads and target writes — no Cloudflare-specific SDK is needed.

**Estimated data volume**: Based on the ~300 LTD products and typical build sizes, the active build data (excluding historical builds not referenced by editions) is estimated at 50–200 GB.
The migration tool should support concurrent transfers with configurable parallelism to complete within a reasonable time window (target: under 4 hours for the full corpus).

**DNS cutover**: The atomic switchover point for Rubin is the DNS change for `*.lsst.io` from Fastly to Cloudflare.
Before the cutover, the migrated content can be verified on a test domain (e.g., `*.lsst-docs-test.org`).
The DNS change is the point of no return — after this, all documentation traffic is served by the Cloudflare Workers stack described in {ref}`documentation-hosting`.
Fastly configuration is retained (but not actively serving) for a rollback window.

### Client migration

Client migration updates the CI workflows that upload documentation builds.
The key difference is that LTD uses per-file presigned URL uploads authenticated with LTD API tokens, while Docverse uses tarball uploads authenticated with Gafaelfawr tokens (see {ref}`projects` and {ref}`github-action`).

The following table summarizes the differences between the current `ltd-upload` action and the Docverse upload path:

| Aspect | `ltd-upload` (current) | `docverse-upload` / updated `ltd-upload` |
|---|---|---|
| Authentication | LTD API username + password (repo or org secret) | Gafaelfawr token (org-level secret `DOCVERSE_TOKEN`) |
| Upload mechanism | Per-file presigned URLs from LTD API | Tarball upload to presigned URL from Docverse API |
| Project identifier | `product` input | `project` input (+ `org` input) |
| API endpoint | `ltd-keeper.lsst.codes` | `docverse.lsst.io` |
| Build registration | `POST /products/{slug}/builds/` | `POST /orgs/{org}/projects/{project}/builds` |

**Key insight**: Many Rubin documentation projects do not call `ltd-upload` directly.
Instead, they use the `lsst-sqre/rubin-sphinx-technote-workflows` reusable workflow (and similar reusable workflows for other document types), which internally calls `ltd-upload`.
Updating the reusable workflow to use the Docverse upload path migrates all downstream projects with **zero per-repo changes**.
Only projects with custom CI workflows that reference `ltd-upload` directly need individual updates.

#### Option A: Retrofit the composite action

Update `lsst-sqre/ltd-upload` (or create a new `docverse-upload` action) to target the Docverse API.
The action accepts a Gafaelfawr token and uploads to Docverse.

The `product` input maps to the Docverse `project`; a new `org` input specifies the organization (defaulting to `rubin` for the Rubin deployment).
GitHub organization-level secrets (`DOCVERSE_TOKEN`) provide the Gafaelfawr token to all repositories in the org without per-repo configuration.

Reusable workflows like `rubin-sphinx-technote-workflows` absorb the interface change — they update their internal `ltd-upload` usage to pass the new inputs, and all downstream repositories that use the reusable workflow migrate automatically.

For repositories with custom workflows that reference `ltd-upload` directly, automated PRs update the workflow YAML (see below).

**Pros**:

- No new service infrastructure to deploy or maintain.
- Clear, explicit upgrade path — each repository's workflow file shows which backend it uses.
- Reusable workflow pattern covers the majority of Rubin repositories with zero per-repo changes.
- Remaining custom-workflow repositories get automated PRs.

**Cons**:

- Repositories with custom workflows need workflow file changes (but this is automatable).
- The `ltd-upload` action name becomes a misnomer once it targets Docverse (mitigated by eventually deprecating it in favor of `docverse-upload`).

#### Option B: LTD API compatibility shim

Deploy a compatibility service at `ltd-keeper.lsst.codes` that translates LTD API calls to Docverse API calls.
Existing workflows continue to call the LTD API endpoints; the shim authenticates LTD credentials and forwards requests to Docverse using a service-level Gafaelfawr token.

**Critical problem: upload format translation.**
The LTD upload flow works as follows:

1. Client calls `POST /products/{slug}/builds/` to register a build.
2. LTD Keeper returns a list of per-file presigned S3 URLs.
3. Client uploads each file directly to S3 using the presigned URLs — these uploads bypass the LTD Keeper API entirely.
4. Client calls `PATCH /builds/{id}` to confirm the upload is complete.

The shim can intercept steps 1, 2, and 4, but **cannot intercept step 3** because the client uploads directly to S3 using presigned URLs.
Docverse expects a single tarball upload, not individual file uploads.
To bridge this gap, the shim would need to either:

- **(a) Replace presigned S3 URLs with shim-hosted upload endpoints**, making the shim a file proxy that receives every file, buffers them, assembles a tarball, and uploads it to Docverse. For a large Sphinx site with thousands of files, this turns the shim into a high-throughput file proxy handling gigabytes of traffic.
- **(b) Monitor the S3 bucket for uploaded files**, detect when all files for a build are present, assemble a tarball, and upload it to Docverse. This is fragile (how to detect "all files uploaded"?) and adds significant latency.

**Pros**:

- Zero workflow changes during the shim's lifetime — existing `ltd-upload` calls work unmodified.

**Cons**:

- The upload format translation (per-file → tarball) makes this architecturally complex. The shim is not a thin API translator — it is a stateful file proxy service.
- Significant development cost for temporary infrastructure that will be decommissioned once migration is complete.
- The shim must handle the full upload throughput of all documentation builds, adding an operational burden (monitoring, scaling, failure handling).
- Development cost is never amortized — the shim is throwaway infrastructure.

#### Recommendation: Option A (retrofit the composite action)

The shim's upload format translation complexity is disproportionate to its benefit.
What appears at first glance to be a thin API translator is actually a stateful file proxy service, because LTD's per-file presigned URL upload pattern cannot be transparently mapped to Docverse's tarball upload pattern without intercepting and buffering all file uploads.

Meanwhile, the reusable workflow pattern means that most Rubin repositories need **zero workflow changes** — updating `rubin-sphinx-technote-workflows` (and similar reusable workflows) migrates all downstream repositories automatically.
The remaining repositories with custom workflows receive automated PRs.
This makes Option A both simpler to implement and less disruptive in practice.

Workflow changes are prepared in advance (PRs opened, reusable workflow branches ready) and merged as part of the cutover maintenance window (see below).

#### Automated PR generation

For repositories with custom workflows that reference `ltd-upload` directly, a migration script scans repositories, identifies `ltd-upload` usage, generates updated workflow YAML, and opens PRs:

1. Use the GitHub API to list repositories in the `lsst` and `lsst-sqre` organizations.
2. For each repository, search `.github/workflows/*.yml` files for references to `lsst-sqre/ltd-upload`.
3. Skip repositories that use reusable workflows (these are migrated by updating the reusable workflow itself).
4. Generate an updated workflow file that replaces the `ltd-upload` step with the new inputs.
5. Open a PR with a standardized description explaining the migration.

**Before** (LTD):

```{code-block} yaml
- name: Upload to LSST the Docs
  uses: lsst-sqre/ltd-upload@v1
  with:
    product: pipelines
    dir: _build/html
  env:
    LTD_USERNAME: ${{ secrets.LTD_USERNAME }}
    LTD_PASSWORD: ${{ secrets.LTD_PASSWORD }}
```

**After** (Docverse via retrofitted action):

```{code-block} yaml
- name: Upload to Docverse
  uses: lsst-sqre/ltd-upload@v2
  with:
    org: rubin
    project: pipelines
    dir: _build/html
    token: ${{ secrets.DOCVERSE_TOKEN }}
```

Or, using the dedicated `docverse-upload` action directly:

```{code-block} yaml
- name: Upload to Docverse
  uses: lsst-sqre/docverse-upload@v1
  with:
    org: rubin
    project: pipelines
    dir: _build/html
    token: ${{ secrets.DOCVERSE_TOKEN }}
```

### Migration strategy: cutover approaches

The data migration and client migration sections above describe _what_ moves and _how_ uploads change.
This section addresses the sequencing — specifically, how to coordinate the DNS cutover (switching `*.lsst.io` from Fastly to Cloudflare) with the CI workflow migration (switching repositories from `ltd-upload` to `docverse-upload`).

The two approaches differ in whether these transitions happen simultaneously or independently.

#### Option A: Coordinated cutover

In this approach, all changes land in a single maintenance window: a final data sync runs, all workflow PRs and reusable workflow updates are merged, and DNS switches to Cloudflare.

| Phase | Description | Key milestone | Rollback strategy |
|---|---|---|---|
| 0: Preparation | Deploy Docverse alongside LTD. Create organizations, configure object stores and CDN, provision Gafaelfawr tokens, validate with test projects. | Docverse deployed and validated with test projects | Remove Docverse deployment; no user impact (LTD unchanged) |
| 1: Data migration | Run `docverse migrate` for all products. Verify migrated content on a test domain. LTD remains authoritative for production traffic. | All active builds and editions migrated and verified | Discard migrated data and re-run; LTD still serving production |
| 2: Client preparation | Prepare all workflow changes without activating them: update `ltd-upload`/`docverse-upload` action, update reusable workflows on branches, open automated PRs for custom-workflow repos. Provision `DOCVERSE_TOKEN` org-level secrets. Test against Docverse staging. | All PRs open and tested; org secrets provisioned | Close PRs; no user impact (LTD unchanged) |
| 3: Cutover | Short maintenance window: run final data sync to capture builds since Phase 1, merge all workflow PRs and reusable workflow changes, switch `*.lsst.io` DNS from Fastly to Cloudflare. New CI builds now flow to Docverse. | Production DNS on Cloudflare, all repos uploading to Docverse | Revert DNS to Fastly and revert workflow merges; Fastly configuration retained for rollback window |

Phase 0 and Phase 1 proceed with no user-visible changes — LTD continues to serve all production traffic.
Phase 2 is preparation — PRs are opened and tested but not merged, so LTD remains fully operational.
Phase 3 is the atomic cutover within a short maintenance window.
Documentation remains readable throughout (LTD serves until DNS propagates).
The window only affects new uploads, which are briefly paused while workflow changes and DNS propagate.
A final data sync at the start of Phase 3 ensures Docverse has all builds up to the cutover moment.
Estimated window: 1–2 hours.

**Tradeoffs**: The coordinated cutover has a simpler overall architecture and a shorter dual-system period — once Phase 3 completes, LTD Keeper can be decommissioned immediately.
However, it requires _all_ CI workflow changes to land simultaneously during the maintenance window.
This is problematic for heterogeneous CI landscapes where projects use custom workflows, Jenkins pipelines, or other non-standard upload patterns that cannot be updated via a reusable workflow change or automated PR.

#### Option B: Sync job with gradual client migration (recommended)

This approach decouples the hosting migration (DNS cutover) from the client migration (CI workflow updates).
DNS switches to Cloudflare early, while LTD Keeper continues running.
A sync job mirrors new builds from LTD into Docverse, allowing projects to migrate their CI workflows individually at their own pace.

The reusable workflow update handles Sphinx technotes automatically — projects using `rubin-sphinx-technote-workflows` migrate with zero per-repo changes when the reusable workflow is updated.
The sync job bridges everything else: LaTeX technotes with custom CI, old reStructuredText technotes with legacy CI patterns, software documentation projects with custom build/upload workflows, and Jenkins-based projects.

Rather than requiring simultaneous PRs to all these project types during a maintenance window, the sync job enables each project team to migrate on their own schedule.
The sync job also serves as a **migration tracking tool**: the set of projects still flowing through the sync path is exactly the set that hasn't been migrated yet, providing clear visibility into migration progress.

##### Revised migration phases

| Phase | Description | Key milestone | Rollback strategy |
|---|---|---|---|
| 0: Preparation | Same as Option A: deploy Docverse, create organizations, configure infrastructure, validate with test projects. | Docverse deployed and validated | Remove Docverse deployment; no user impact |
| 1: Data migration | Same as Option A: run `docverse migrate` for all products, verify on test domain. | All active builds and editions migrated and verified | Discard migrated data and re-run; LTD still serving |
| 2: DNS cutover | Switch `*.lsst.io` DNS from Fastly to Cloudflare. Deploy and start the sync job. Update the reusable workflow and merge automated PRs for projects that are ready. No requirement for all projects to migrate simultaneously. | Production DNS on Cloudflare; sync job running | Revert DNS to Fastly; stop sync job |
| 3: Gradual client migration | Projects migrate their CI workflows individually. The sync job handles projects still uploading to LTD. Monitor the sync job dashboard to track migration progress. | All projects migrated to native Docverse uploads | Sync job continues to bridge any remaining projects |
| 4: Decommission | Stop the sync job. Shut down LTD Keeper. Remove LTD infrastructure. | LTD Keeper decommissioned | N/A (all projects on Docverse) |

Phase 2 replaces the coordinated cutover with a DNS-only switch.
Because the sync job mirrors LTD builds into Docverse, documentation remains continuously available — projects that haven't migrated their CI continue to upload to LTD, and the sync job ensures those builds appear in Docverse within seconds.

Phase 3 is an extended period (weeks to months) during which project teams migrate at their own pace.
The reusable workflow update migrates the majority of Sphinx technotes immediately.
The sync job bridges the remaining projects until they complete their individual migrations.

##### Sync job design

The sync job is a polling service that discovers new builds in LTD, copies their objects to the Docverse object store, and registers them via the Docverse API.

```{mermaid}
sequenceDiagram
    participant Sync as Sync Job
    participant LTD_API as LTD Keeper API
    participant S3 as LTD S3 Bucket
    participant DV_API as Docverse API
    participant R2 as Docverse R2 Bucket
    participant KV as CDN Edge KV

    loop Every 60 seconds
        Sync->>LTD_API: GET /products/ (list all products)
        LTD_API-->>Sync: Product list

        loop For each synced product
            Sync->>LTD_API: GET /products/{slug}/builds/?uploaded=true
            LTD_API-->>Sync: Builds since last checkpoint

            loop For each new build
                Sync->>S3: List objects at {product}/builds/{slug}/
                S3-->>Sync: Object keys + metadata

                loop For each object
                    Sync->>R2: PUT {project}/__builds/{build_id}/{file}
                end

                Sync->>DV_API: Register build (metadata, object inventory)
                DV_API-->>Sync: Build created
            end

            Sync->>LTD_API: GET /products/{slug}/editions/
            LTD_API-->>Sync: Edition states

            loop For each edition with updated build pointer
                Sync->>DV_API: Update edition → build mapping
                DV_API->>KV: Write edition→build mapping
            end
        end
    end
```

**Discovery**: The sync job polls the LTD API for builds where `uploaded=true` with a `date_created` after the last checkpoint timestamp.
Each polling cycle processes only new builds, making the steady-state cost proportional to the build rate rather than the total build count.

**Build identity mapping**: The sync job maintains a mapping of `(product_slug, ltd_build_slug)` → `docverse_build_id`.
This mapping is stored in the Docverse database (or a dedicated sync state table) and ensures that repeated syncs are idempotent — a build that has already been synced is skipped.

**Object transfer**: Objects are copied directly from `{product}/builds/{build_id}/` in the LTD S3 bucket to `{project}/__builds/{build_id}/` in the Docverse R2 bucket.
This bypasses the normal Docverse tarball upload flow (tarball → unpack → inventory) because the objects are already unpacked in the LTD bucket.
The sync job constructs the `BuildObject` inventory by listing the copied objects and recording their keys, content hashes, content types, and sizes.
The build is then registered via the Docverse API with the pre-populated inventory.

**Edition sync**: After syncing builds, the sync job checks each edition's `build_id` and `date_rebuilt` fields in the LTD API.
If an edition's build pointer has changed since the last sync, the sync job updates the corresponding Docverse edition to point to the newly synced build.

**Project ownership model**: Each project in Docverse has a sync state that is one of:

```{mermaid}
stateDiagram-v2
    [*] --> SyncedFromLTD: Project created by migration tool
    SyncedFromLTD --> NativeDocverse: Project team migrates CI workflow
    NativeDocverse --> [*]: Steady state

    state SyncedFromLTD {
        [*] --> Polling
        Polling --> CopyingBuild: New build detected
        CopyingBuild --> UpdatingEditions: Build registered
        UpdatingEditions --> Polling: Editions updated
    }
```

- **SyncedFromLTD**: The sync job actively mirrors builds from LTD for this project. The project's CI workflows still upload to LTD Keeper.
- **NativeDocverse**: The project's CI workflows upload directly to Docverse. The sync job ignores this project.

The transition from `SyncedFromLTD` to `NativeDocverse` happens when the project team merges their CI workflow update.
A Docverse API endpoint (or CLI command) flips the project's sync state, and the sync job stops polling LTD for that project.

##### Sync latency and consistency

A 60-second polling interval adds negligible latency to the documentation publish pipeline.
Documentation builds already take minutes (Sphinx builds, CI pipeline overhead), so an additional 60-second delay before the build appears on `*.lsst.io` is imperceptible to users.

There is a brief consistency window after a build completes in LTD but before the sync job copies it to Docverse.
During this window, the LTD API shows the build as complete, but the Cloudflare-served site has not yet been updated.
This window is at most one polling interval (60 seconds) plus the time to copy objects — typically a few seconds for a standard documentation build.

##### Two sources of truth

The sync job creates a period where both LTD Keeper and Docverse contain build data for the same projects.
The project ownership model (described above) prevents conflicts:

- The sync job only writes to projects in the `SyncedFromLTD` state. It never modifies projects that have transitioned to `NativeDocverse`.
- Direct Docverse uploads are rejected for projects in the `SyncedFromLTD` state (or at minimum, logged as warnings), preventing accidental dual-writes.
- Project owners are notified when their project is in the synced state and given instructions for migrating their CI workflow.

#### Comparison of approaches

| Aspect | Option A: Coordinated cutover | Option B: Sync job |
|---|---|---|
| Maintenance window | 1–2 hours; all changes land simultaneously | Minutes for DNS cutover only |
| CI migration timing | All projects migrate during the window | Projects migrate individually over weeks/months |
| Dual-system duration | Minimal (LTD decommissioned after Phase 3) | Extended (LTD runs until all projects migrate) |
| Architecture complexity | Lower (no sync job) | Higher (sync job + project ownership model) |
| Risk during cutover | Higher (many simultaneous changes) | Lower (DNS-only change, sync provides continuity) |
| Development effort | Lower (no sync job to build) | Moderate (sync job is straightforward ETL, not a complex service) |
| Ongoing S3 egress | One-time during data migration | Continuous during sync period (proportional to build rate) |
| Migration visibility | Binary (all migrated or not) | Per-project tracking via sync state |
| Non-standard CI support | Requires all custom workflows ready before cutover | Custom workflows migrate on their own schedule |

#### Recommendation

**The sync job approach (Option B) is recommended for the Rubin deployment.**

While Sphinx technotes using `rubin-sphinx-technote-workflows` migrate automatically via the reusable workflow update, many project types do not benefit from this pattern:

- **LaTeX technotes** — custom CI workflows, no reusable workflow
- **Old reStructuredText technotes** — may use legacy CI patterns
- **Software documentation projects** (e.g., `pipelines.lsst.io`) — custom build and upload workflows
- **Jenkins-based projects** — entirely separate CI system

Requiring all of these projects to have PRs ready and merged during a 1–2 hour maintenance window is operationally risky and logistically complex.
The sync job avoids this by letting each project team migrate on their own schedule, while the sync job ensures continuous documentation availability.

The sync job is also simpler than it might appear at first glance.
Unlike the rejected API compatibility shim (Option B in the client migration section), the sync job does not need to proxy uploads or translate between upload formats in real time.
It reads completed builds from LTD — objects that are already unpacked and stored in S3 — and copies them to R2.
This is straightforward ETL, not a stateful proxy service.

The coordinated cutover (Option A) remains a viable alternative for smaller deployments where all projects use the reusable workflow pattern and can be migrated simultaneously.

### Risk mitigation

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| Data corruption during migration | Incorrect documentation served | Low | Verify migrated content against LTD originals using content hash comparison; test domain validation before DNS cutover |
| DNS cutover causes outage | Documentation unavailable | Low | Test DNS configuration in advance; retain Fastly configuration for rapid rollback; use low TTL during cutover window |
| Incomplete build migration | Missing pages or broken links | Medium | Migration tool validates object counts per build against LTD inventory; flag discrepancies for manual review |
| Gafaelfawr token provisioning issues | CI uploads fail | Medium | Provision and test org-level secrets during Phase 2 preparation; validate with test uploads before cutover |
| Builds in flight during cutover | Some CI builds fail or upload to wrong backend | Low | Announce maintenance window in advance; re-trigger any builds that fail during the cutover window |
| Reusable workflow update breaks downstream repos | CI failures across many repos | Low | Test reusable workflow changes against a representative sample of downstream repos before merging; reusable workflow versioning allows rollback |
| Sync job latency | Builds visible in LTD before Docverse | Low | 60-second polling interval keeps latency well within acceptable bounds; monitoring alerts if sync falls behind |
| Dual source of truth conflicts | Inconsistent documentation state | Low | Project ownership model enforces single-writer semantics; sync job only writes to `SyncedFromLTD` projects; direct uploads blocked for synced projects |
| Sync job failure | New LTD builds not mirrored to Docverse | Medium | Health check endpoint and monitoring; dead-letter queue for failed syncs; manual re-sync capability; LTD content remains accessible as fallback during sync outage |
| Extended S3 egress costs | Higher AWS costs during sync period | Low | Costs are proportional to build rate (not total data volume); most builds are small (< 100 MB); monitor egress and set billing alerts |
