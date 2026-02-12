(projects)=

## Projects, editions and builds

The key domain entities from LTD are retained in Docverse.

- **Project**: a documentation site with a stable URL and multiple versions (editions). Projects are owned by organizations and have metadata like name, description, and configuration.
- **Edition**: a published version of a project, representing a specific build's content at a stable URL. Editions have tracking modes that determine which builds they follow (e.g., `main` edition tracks the default branch, `DM-12345` edition tracks branches matching `tickets/DM-12345`).
- **Build**: a discrete upload of documentation content for a project. Builds are conceptually immutable and carry metadata about their origin (git ref, uploader identity, etc). Builds are identified with a Crockford Base32 ID.

### Project model simplification

In the original LTD API, projects were called "products" to borrow terminology from EUPS.
With time, we realized that EUPS wasn't relevant to LTD, and we shifted the terminology to "project."

With Docverse, we further improve projects by removing all infrastructure configuration from the project model and moving it to the organization level. This simplifies the project model and reflects the fact that infrastructure configuration (e.g., object store settings, CDN settings) is typically shared across all projects within an organization. See the {ref}`organizations` section for details on the new organization model and how infrastructure configuration is handled there.

### Improved build uploads

An issue with the LTD API was that build uploads are slow for large documentation sites.
This was because the client uploaded each file individually using presigned URLs.

Docverse uses a **tarball-to-object-store** upload model. The client compresses the built documentation into a tarball, uploads it directly to the object store via a presigned URL, then signals Docverse to process it. This avoids the performance problems of per-file presigned URLs (HTTP overhead per file, no compression, thousands of separate uploads for large Sphinx sites) and keeps the API server thin by never routing large request bodies through the API.

#### End-to-end flow

This sequence diagram illustrates the end-to-end flow of a build upload and processing:

```{mermaid}
sequenceDiagram
    participant Client
    participant API as Docverse API
    participant Store as Object Store
    participant Worker as Docverse Worker

    Client->>API: Authenticate (Gafaelfawr token)
    Client->>API: POST create build (git ref, content hash)
    API-->>Client: Presigned upload URL
    Client->>Store: Upload tarball (PUT via presigned URL)
    Client->>API: PATCH signal upload complete
    API-->>Client: queue_url for tracking
    API->>Worker: Enqueue build processing job
    Worker->>Store: Download tarball from staging
    Worker->>Store: Stream-unpack & upload files to build prefix
    Worker->>Store: Delete staging tarball
    Worker->>Worker: Inventory objects in Postgres, evaluate tracking rules
    Worker->>Store: Update editions (pointer or copy mode)
    Worker->>Worker: Render project dashboard
```

1. **Client authenticates** with a Gafaelfawr token (org uploader role).
2. **Client creates a build record** via the API (`POST /orgs/:org/projects/:project/builds`). The request includes the git ref and a content hash of the tarball. The response provides a single presigned upload URL pointing to the staging location. See the {ref}`api` section for request/response details.
3. **Client uploads the tarball** directly to the object store using the presigned URL. The tarball is a gzipped tar archive (`.tar.gz`) of the built documentation directory. The client implementation is straightforward -- `tar czf` piped to an HTTP PUT. The object store handles the bandwidth; the API server is not involved. Multipart uploads are supported where the object store provider allows (S3, GCS), enabling resumable uploads for large sites.
4. **Client signals upload complete** (`PATCH /orgs/:org/projects/:project/builds/:build` with status update). The response includes a `queue_url` for tracking the background processing.
5. **Background processing** -- a single oban-py job executes the build processing pipeline:
   1. **Download tarball** from the staging location (staging bucket or `__staging/` prefix in the publishing bucket).
   2. **Stream-unpack and upload**: extract entries from the tar stream and upload individual files to the build's permanent prefix (`__builds/{build_id}/`) in the publishing bucket. Uploads are parallelized via an `asyncio.Semaphore`-bounded pool of concurrent uploads. For a 5,000-file Sphinx site, this processes in well under a minute.
   3. **Delete staging tarball** after successful extraction.
   4. **Inventory** the build's objects in Postgres.
   5. **Evaluate tracking rules** to determine affected editions.
   6. **Update editions** in parallel via `asyncio.gather()`.
   7. **Render project dashboard and metadata JSON** once after all edition updates.

The API handler for step 4 is thin: it validates the request, updates the build status to `processing`, enqueues the single oban job, and returns the `queue_url`.

#### Staging location

The tarball is uploaded to a staging location that is separate from the build's permanent prefix. The staging path is `__staging/{build_id}.tar.gz`. Where this lives depends on whether the org has a dedicated staging store configured:

- **With staging store**: the presigned URL points to the staging bucket. The worker reads from the staging bucket (fast, intra-region) and writes extracted files to the publishing bucket.
- **Without staging store**: the presigned URL points to the publishing bucket using the `__staging/` prefix. The worker reads and writes within the same bucket.

The staging store optimization matters when the publishing bucket is on a different network than the Docverse compute cluster. For example, in Rubin Observatory's deployment:

- **Staging bucket**: GCS in `us-central1` (same region as the GKE cluster). The CI runner uploads the tarball to GCS. The Docverse worker downloads it over Google's internal network -- fast, with zero egress cost.
- **Publishing bucket**: Cloudflare R2. The worker uploads extracted files to R2, which is optimized for CDN serving with zero egress cost to readers.

Without the split, the worker would download the tarball from R2 over the public internet, unpack it, then upload thousands of individual files back to R2 over the public internet. With the split, only the final extracted files cross the network boundary, and the tarball round-trip stays within GCP.

For orgs where the publishing store is in the same region as the cluster (e.g., GCS publishing via Google Cloud CDN, or S3 publishing via CloudFront in the same AWS region), the staging store adds no benefit and can be left unconfigured.

#### Stream unpacking

The worker unpacks the tarball using Python's `tarfile` module in streaming mode, uploading each file to the publishing bucket as it's extracted. This keeps memory usage bounded -- the worker never needs to hold the entire unpacked site in memory or on disk:

```{code-block} python
async def unpack_and_upload(
    self,
    staging_store: ObjectStore,
    publishing_store: ObjectStore,
    build_id: str,
    semaphore: asyncio.Semaphore,
) -> list[str]:
    """Stream-unpack a tarball and upload files to the build prefix."""
    tarball_stream = await staging_store.get_object_stream(
        f"__staging/{build_id}.tar.gz"
    )
    uploaded_keys: list[str] = []
    upload_tasks: list[asyncio.Task] = []

    with tarfile.open(fileobj=tarball_stream, mode="r:gz") as tar:
        for member in tar:
            if not member.isfile():
                continue
            file_obj = tar.extractfile(member)
            if file_obj is None:
                continue
            content = file_obj.read()
            key = f"__builds/{build_id}/{member.name}"

            async def _upload(k: str, data: bytes) -> None:
                async with semaphore:
                    await publishing_store.upload_object(
                        key=k,
                        data=data,
                        content_type=guess_content_type(k),
                    )

            task = asyncio.create_task(_upload(key, content))
            upload_tasks.append(task)
            uploaded_keys.append(key)

    await asyncio.gather(*upload_tasks)
    await staging_store.delete_object(f"__staging/{build_id}.tar.gz")
    return uploaded_keys
```

The semaphore bounds concurrency (e.g., 50 concurrent uploads) to avoid overwhelming the object store API. Content types are inferred from file extensions.

### Build object inventory table

Each build has an associated set of object records in Postgres: key (object path), content hash (ETag or SHA-256), content type, and size. This is populated during the inventory phase of the build processing job. The inventory enables:

- Fast diff computation for edition updates (no object store listing calls)
- Orphan detection for build cleanup rules
- Metadata for dashboards (e.g., build size)

Note that this inventory table is motivated for the original S3 and Fastly-based architecture where editions are updated by copying the build objects. With the Cloudflare-based architecture where editions are updated by pointing to the build prefix, the inventory is less critical for edition updates but still be valuable for orphan detection and dashboard metadata.

### Edition overview

An Edition is a named, published view of a Product's documentation at a stable URL. Editions are pointers — they represent a specific build's content served at an edition-specific URL path (e.g., `/v/main/`, `/v/DM-12345/`, `/v/2.x/`).

Two concepts govern edition behavior:

- **Tracking mode**: determines _which builds_ the edition follows (the algorithm for auto-updating).
- **Edition kind**: classifies the edition for dashboard display and lifecycle rule targeting.

### Edition slugs

Editions are identified by URL-safe slugs. The slug system has three layers:

1. **Reserved slugs**: `__main` is the sole reserved slug, representing the default edition that serves at the project root (no `/v/` prefix in the URL). It does not correspond to a git ref and uses double-underscore prefix to avoid collisions with any git branch or tag name.

2. **Org-configurable rewrite rules**: organizations can configure pattern-based transforms from git ref → edition slug. These are an ordered list of rules where the first match wins, with three rule types: `prefix_strip`, `regex`, and `ignore`. For example, Rubin's convention uses a `prefix_strip` rule to rewrite `tickets/DM-12345` → slug `DM-12345`. See the {ref}`edition-slug-rewrite-rules` section for the full rule format, evaluation algorithm, and examples.

3. **Default behavior**: slashes in git refs become dashes (e.g., `feature/dark-mode` → `feature-dark-mode`), keeping all edition slugs as single URL path segments.

An edition tracks a **slug**, not a single canonical git ref. Multiple git refs can map to the same slug through rewrite rules and all contribute builds to that edition. For example, if both `tickets/DM-12345` and `DM-12345` exist as branches and both rewrite to slug `DM-12345`, builds from either ref update the same edition. This fixes a bug in LTD Keeper where only one ref could contribute to a special-case edition.

(edition-slug-rewrite-rules)=

### Edition slug rewrite rules

When a new build arrives and its git ref doesn't match any existing edition, Docverse uses **slug rewrite rules** to determine how to transform the git ref into an edition slug. Rules are evaluated in order; the first match wins.

#### Rule types

Three rule types cover the practical use cases:

**`prefix_strip`** — the workhorse. Matches refs starting with a literal prefix, strips it, and uses the remainder as the slug (with any remaining slashes replaced by a configurable character, default `-`).

```json
{
  "type": "prefix_strip",
  "prefix": "tickets/",
  "edition_kind": "draft"
}
```

`tickets/DM-12345` → slug `DM-12345`. `tickets/foo/bar` → slug `foo-bar`.

**`regex`** — for patterns that prefix stripping can't express. Uses a Python regex with a named capture group `slug` to extract the edition slug. Remaining slashes in the captured group are still replaced by `slash_replacement` by default, protecting against invalid slugs.

```json
{
  "type": "regex",
  "pattern": "^release/(?P<slug>v\\d+\\.\\d+)$",
  "edition_kind": "release"
}
```

`release/v2.3` → slug `v2.3` with kind `release`. `release/experimental` → no match.

**`ignore`** — suppresses edition auto-creation for matching refs. Uses glob patterns (Python `fnmatch` semantics with `**` for recursive matching). Useful for filtering noise from dependency bot branches, CI scratch branches, and similar refs that should never produce editions.

```json
{
  "type": "ignore",
  "glob": "dependabot/**"
}
```

`dependabot/npm/lodash-4.17.21` → no edition created.

#### Rule fields

| Field               | Type | Applies to              | Default   | Description                                                                           |
| ------------------- | ---- | ----------------------- | --------- | ------------------------------------------------------------------------------------- |
| `type`              | enum | all                     | —         | `prefix_strip`, `regex`, or `ignore`                                                  |
| `edition_kind`      | str  | `prefix_strip`, `regex` | `"draft"` | Kind assigned to auto-created editions                                                |
| `prefix`            | str  | `prefix_strip`          | —         | Literal prefix to match and strip                                                     |
| `pattern`           | str  | `regex`                 | —         | Python regex with named group `slug`                                                  |
| `glob`              | str  | `ignore`                | —         | Glob pattern for ref matching                                                         |
| `slash_replacement` | str  | `prefix_strip`, `regex` | `"-"`     | Character replacing remaining slashes in extracted slug. Must be one of `-`, `_`, `.` |

#### Storage and scoping

Rules are stored as a JSONB array on the Organization and Project tables:

- **`Organization.slug_rewrite_rules`** (JSONB): ordered rule list applied to all projects in the org.
- **`Project.slug_rewrite_rules`** (JSONB, nullable): when set, **completely replaces** the org-level rules for that project. No merging or inheritance — if a project needs one different rule, it copies the org rules and modifies. This avoids the complexity of ordered-list merge semantics.

Setting the project-level rules to `null` (or omitting the field) restores inheritance from the org.

#### Evaluation algorithm

```
1. rules = project.slug_rewrite_rules ?? org.slug_rewrite_rules ?? []
2. For each rule in order:
   a. If type=ignore and glob matches git_ref → return None (suppress)
   b. If type=prefix_strip and git_ref starts with prefix →
      remainder = git_ref[len(prefix):]
      slug = remainder.replace("/", rule.slash_replacement)
      return (slug, rule.edition_kind)
   c. If type=regex and pattern matches git_ref →
      slug = match.group("slug")
      slug = slug.replace("/", rule.slash_replacement)
      return (slug, rule.edition_kind)
3. No rule matched (default fallback):
   slug = git_ref.replace("/", "-")
   return (slug, "draft")
```

The default fallback (step 3) always applies, so every non-ignored ref produces a valid slug even with zero rules configured. This preserves LTD Keeper's existing behavior for orgs that don't configure any rewrite rules.

#### Slug validation

After a slug is produced (by rule or default), it is validated:

- Must be non-empty.
- Must contain only URL-safe characters: lowercase alphanumeric, hyphens, underscores, dots. Uppercase characters are lowercased.
- Must not start with `__` (reserved prefix for system slugs like `__main`).
- Must not exceed 128 characters.

If validation fails, the build is processed but no edition is auto-created. The build record's status reflects that slug generation failed, and the issue is logged for operator attention.

#### Example: Rubin Observatory configuration

```json
[
  { "type": "ignore", "glob": "dependabot/**" },
  { "type": "ignore", "glob": "renovate/**" },
  { "type": "prefix_strip", "prefix": "tickets/", "edition_kind": "draft" },
  {
    "type": "regex",
    "pattern": "^v?(?P<slug>\\d+\\.\\d+\\.\\d+)$",
    "edition_kind": "release"
  }
]
```

Evaluation for various git refs:

| Git ref                         | Matched rule             | Slug                   | Kind      |
| ------------------------------- | ------------------------ | ---------------------- | --------- |
| `dependabot/npm/lodash-4.17.21` | `ignore` (index 0)       | — (suppressed)         | —         |
| `renovate/typescript-5.x`       | `ignore` (index 1)       | — (suppressed)         | —         |
| `tickets/DM-12345`              | `prefix_strip` (index 2) | `DM-12345`             | `draft`   |
| `tickets/DM-99999`              | `prefix_strip` (index 2) | `DM-99999`             | `draft`   |
| `v2.3.0`                        | `regex` (index 3)        | `2.3.0`                | `release` |
| `2.3.0`                         | `regex` (index 3)        | `2.3.0`                | `release` |
| `feature/dark-mode`             | default fallback         | `feature-dark-mode`    | `draft`   |
| `main`                          | default fallback         | `main`                 | `draft`   |

Note: for `main`, the default fallback produces slug `main` with kind `draft`, but in practice the `__main` edition already exists and matches via its tracking mode, so auto-creation is not triggered.

#### Dry-run endpoint

A preview endpoint allows org admins to test their rewrite rules against a git ref without creating any resources:

```
POST /orgs/:org/slug-preview                         → preview slug resolution (admin)
```

Request:

```json
{
  "git_ref": "tickets/DM-12345",
  "project": "pipelines"
}
```

The optional `project` field causes the endpoint to use project-level rule overrides if they exist. Without it, the org-level rules are used.

Response:

```json
{
  "git_ref": "tickets/DM-12345",
  "edition_slug": "DM-12345",
  "edition_kind": "draft",
  "matched_rule": {
    "type": "prefix_strip",
    "prefix": "tickets/",
    "index": 0
  },
  "rule_source": "org"
}
```

For an ignored ref:

```json
{
  "git_ref": "dependabot/npm/lodash-4.17.21",
  "edition_slug": null,
  "edition_kind": null,
  "matched_rule": {
    "type": "ignore",
    "glob": "dependabot/**",
    "index": 0
  },
  "rule_source": "org"
}
```

For a ref that hits the default fallback:

```json
{
  "git_ref": "feature/dark-mode",
  "edition_slug": "feature-dark-mode",
  "edition_kind": "draft",
  "matched_rule": null,
  "rule_source": "default"
}
```

The `rule_source` field indicates whether the rules came from `"org"`, `"project"`, or `"default"` (no rules configured, using built-in fallback).

### Tracking modes

The full set of tracking modes in Docverse:

| Mode                  | Behavior                                                                                                                          | Carried from                   |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| `git_ref`             | Track a specific branch or tag                                                                                                    | LTD Keeper v1 (was `git_refs`) |
| `lsst_doc`            | Track latest LSST document version tag (`vMajor.Minor`)                                                                           | LTD Keeper v1                  |
| `eups_major_release`  | Track latest EUPS major release tag                                                                                               | LTD Keeper v1                  |
| `eups_weekly_release` | Track latest EUPS weekly release tag                                                                                              | LTD Keeper v1                  |
| `eups_daily_release`  | Track latest EUPS daily release tag                                                                                               | LTD Keeper v1                  |
| `semver_release`      | Track the latest semver release, excluding pre-releases (alpha, beta, rc)                                                         | New                            |
| `semver_major`        | Track the latest release within a major version stream (e.g., latest `2.x.x`). Parameterized by `major_version`.                  | New                            |
| `semver_minor`        | Track the latest release within a minor version stream (e.g., latest `2.3.x`). Parameterized by `major_version`, `minor_version`. | New                            |

Semver tracking supports tags both with and without a `v` prefix (e.g., `v2.1.0` and `2.1.0`).

### Edition kinds

Edition kinds classify editions for display and lifecycle purposes:

| Kind      | Description                   | Typical tracking mode               |
| --------- | ----------------------------- | ----------------------------------- |
| `main`    | The default edition           | `git_ref` tracking main/master      |
| `release` | A stable release              | `semver_release`, `lsst_doc`        |
| `draft`   | A draft/development edition   | `git_ref` tracking a feature branch |
| `major`   | Tracks a major version stream | `semver_major`                      |
| `minor`   | Tracks a minor version stream | `semver_minor`                      |

When an edition is auto-created, its kind is assigned based on the tracking mode. The kind does not constrain tracking behavior — it provides context for dashboards and lifecycle rules.

### Auto-creation of editions

When a new build arrives and matches no existing edition's tracking criteria, Docverse can auto-create editions:

- **git_ref editions**: auto-created for new branches/tags (as in LTD Keeper). Classified as `draft` kind by default.
- **semver_major editions**: auto-created when a build introduces a new major version stream (e.g., first `v3.x.x` tag). Classified as `major` kind.
- **semver_minor editions**: auto-created when a build introduces a new minor version stream (e.g., first `v3.1.x` tag). Classified as `minor` kind.

Auto-creation for semver_major and semver_minor modes can be disabled at both the org and project level.

### Build annotations

The upload client can optionally annotate a build with metadata about its nature (e.g., "this is a release", "this is a draft/PR build"). This supplements the pattern-based classification from tracking rules. Two complementary mechanisms:

- **Client annotations**: optional metadata on the build record provided at upload time.
- **Project/org pattern rules**: configurable rules that classify builds based on git ref patterns (e.g., "tags matching `v*.*.*` are releases", "branches matching `tickets/*` are drafts").

The tracking system uses both signals, with pattern-based rules as the primary classifier.

### Edition-build history

Docverse maintains an explicit log of every build that an edition has pointed to, stored in an `EditionBuildHistory` table with the edition ID, build ID, timestamp, and ordering position. This replaces the implicit relationship in LTD Keeper (where you'd have to reconstruct history from build timestamps). The history enables:

- **Rollback API**: an org admin can roll an edition back to any previous build in its history with a single API call (PATCH the edition with a `build` field pointing to the desired build).
- **Orphan build detection**: lifecycle rules can reference history position (e.g., "a build that is 5+ versions back and older than 30 days is an orphan").

(edition-update-strategy)=
### Edition update strategy

When an edition is updated to point to a new build, the strategy depends on the CDN's declared capabilities — **pointer mode** or **copy mode**.

#### Pointer mode (instant switchover)

For CDNs with edge compute and an edge data store (see {ref}`cdn-provider-evaluation`), the edition is a metadata pointer and no object copying is required. The update process:

1. **Write new mapping**: update the edition→build mapping in the edge data store (e.g., Cloudflare Workers KV, Fastly KV Store).
2. **Purge CDN cache**: invalidate cached content for the edition so new requests resolve the updated mapping.
3. **Update database**: record the new edition→build association and log to `EditionBuildHistory`.

The edge Worker/Compute function intercepts each request, looks up the current build for the requested edition in the edge data store, and fetches the content directly from the build's object store prefix. Edition updates are effectively instant — a KV write + cache purge, no bulk object operations.

#### Copy mode (ordered in-place update)

For CDNs without edge compute, Docverse performs an ordered in-place update (the fallback from the LTD Keeper approach of delete-then-copy, which caused temporary 404s):

1. **Diff**: compare the new build's object inventory against the edition's current inventory using the Postgres object tables.
2. **Copy new/changed assets first**: images, CSS, JS — so that HTML pages referencing new assets won't break.
3. **Copy HTML pages**: starting from the deepest directory levels, working up to the homepage last. This minimizes the window where a user might see a partially-updated site.
4. **Move orphaned objects to purgatory**: objects in the edition that don't exist in the new build are moved to a purgatory key prefix rather than deleted immediately.
5. **Purge CDN cache**: invalidate cached content for the edition.

This ordering ensures that at no point during the update will a user get a 404 for content that should exist.

#### Mode selection

The edition update service checks the CDN's declared `supports_pointer_mode` capability at runtime to select the appropriate code path. This keeps the service logic clean and makes it straightforward to add new CDN providers without modifying orchestration logic. Orgs using Cloudflare Workers + R2 or Fastly Compute get instant switchovers; orgs using Google Cloud CDN or other providers without edge compute get the ordered copy strategy.

### Deletion and lifecycle rules

#### Soft delete and purgatory

Docverse uses a two-layer soft delete approach:

- **Database**: objects are soft-deleted (marked with a `date_ended` or `date_deleted` timestamp) rather than immediately removed.
- **Object store**: files are moved to a purgatory key prefix rather than deleted. A background job hard-deletes purgatory objects after a configurable retention period.

This provides reversibility at both layers. The purgatory timeout is configurable at the org level with per-project overrides.

#### Lifecycle rules

Lifecycle rules are stored as JSONB in Postgres — a list of rule objects, each with a `type` discriminator and type-specific parameters. Rules are configured at the org level as defaults and can be overridden per-project. Individual editions can be marked as exempt from all lifecycle rules (a "never delete" flag).

Rule types:

| Rule type              | Parameters                     | Behavior                                                                               |
| ---------------------- | ------------------------------ | -------------------------------------------------------------------------------------- |
| `draft_inactivity`     | `max_days_inactive`            | Delete draft editions with no new builds for N days                                    |
| `ref_deleted`          | `enabled`                      | Delete editions whose tracked Git ref no longer exists on GitHub                       |
| `build_history_orphan` | `min_position`, `min_age_days` | Delete builds that are N+ positions back in an edition's history and older than M days |

Example rule configuration (JSONB):

```json
[
  { "type": "draft_inactivity", "max_days_inactive": 30 },
  { "type": "ref_deleted", "enabled": true },
  { "type": "build_history_orphan", "min_position": 5, "min_age_days": 30 }
]
```

Different edition kinds can have different default lifecycle rules. For example, `draft` editions might default to `draft_inactivity` with 30 days, while `release` editions have no auto-deletion rules.

#### GitHub event-driven deletion

Edition deletion triggered by Git ref deletion is event-driven via GitHub webhooks. Docverse supports two models for receiving GitHub events:

- **Direct webhooks**: Docverse receives webhook events directly as a GitHub App, using the Safir GitHub App framework.
- **Kafka via Squarebot**: Squarebot acts as a GitHub App events gateway, receiving webhooks and republishing them internally via Kafka. Docverse consumes events from Kafka. This allows Docverse to share a GitHub App installation with other internal tools.

Both models are supported; the deployment can use either or both.

A periodic **audit job** supplements the event-driven approach by verifying that Git refs referenced by Docverse editions and projects still exist on GitHub. This catches cases where webhook delivery failed or events were missed.
