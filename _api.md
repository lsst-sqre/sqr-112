(api)=

## REST API design

### Conventions

The Docverse REST API follows SQuaRE's REST API design conventions, and specifically follows patterns established in [Ook](https://github.com/lsst-sqre/ook) and [Times Square](https://github.com/lsst-sqre/times-square):

- **No URL versioning**: the API has a single set of paths (no `/v1/` prefix). Breaking changes would be handled by introducing new endpoints alongside deprecated ones. A lot of design work would have to go into supporting multiple API versions simultaneously, so assuming path-based API versioning seems presumptive at this stage.
- **HATEOAS-style navigation**: all resource representations include `self_url` and relevant navigation URLs (e.g., `project_url`, `org_url`, `builds_url`, `editions_url`). Clients navigate the API via these provided URLs rather than constructing their own.
- **Collections are top-level arrays**: collection endpoints return a JSON array of resource objects. Pagination metadata is carried in response headers, not a wrapper object.
- **Keyset pagination**: all collection endpoints use keyset pagination via [Safir's pagination library](https://safir.lsst.io/user-guide/database/pagination.html). Pagination cursors and links are returned in headers.
- **Errors**: all error responses follow `safir.models.ErrorModel` — a `detail` array of `{type, msg, loc?}` objects, compatible with FastAPI's built-in validation error format.
- **Background job URLs**: when a POST or PATCH enqueues background work (e.g., build processing, edition updates), the response includes a `queue_url` pointing to the job status resource.

### Resource identifiers

Resources use human-readable identifiers in URLs rather than auto-generated database primary keys:

| Resource       | URL identifier      | Format                                           | Example                                     |
| -------------- | ------------------- | ------------------------------------------------ | ------------------------------------------- |
| Organization   | slug                | lowercase alphanumeric + hyphens                 | `rubin`, `spherex`                          |
| Project        | slug                | URL-safe, matches doc handle                     | `sqr-006`, `pipelines`                      |
| Edition        | slug                | URL-safe, derived from git ref via rewrite rules | `__main`, `DM-12345`, `v2.x`                |
| Build          | Crockford Base32 ID | 12 chars + checksum, hyphenated                  | `01HQ-3KBR-T5GN-8W`                         |
| Queue job      | Crockford Base32 ID | 12 chars + checksum, hyphenated                  | `0G4R-MFBZ-K7QP-5X`                         |
| Org membership | composite key       | `{principal_type}:{principal}`                   | `user:docverse-ci-rubin`, `group:g_spherex` |

Build and queue job IDs use the [Crockford Base32 implementation from Ook](https://github.com/lsst-sqre/ook/blob/main/src/ook/domain/base32id.py), backed by `base32-lib`. IDs are stored as integers in Postgres but serialized as base32 strings with checksums in the API via Pydantic. Build IDs are randomly generated (not ordered). Queue job IDs are Docverse-owned public identifiers that map internally to the queue backend's job IDs via the `backend_job_id` column in the `QueueJob` table.

### Ingress and authorization mapping

Two Gafaelfawr ingresses protect the API:

| Ingress path | Required scope        | Purpose                                    |
| ------------ | --------------------- | ------------------------------------------ |
| `/admin/*`   | `admin:docverse`      | Superadmin operations (create/delete orgs) |
| `/*`         | `exec:docverse`       | All other operations                       |

All org-level authorization (admin vs uploader vs reader) is enforced at the **application layer** via `OrgMembership` checks. Gafaelfawr ingresses cannot express "user X has role Y in org Z" — that granularity requires application-level logic. The ingress layer ensures the user is authenticated and has basic Docverse access; the application layer checks the specific role required for each endpoint.

### Endpoint catalog

#### Root

```
GET  /                                              → API metadata and navigation URLs
```

Returns API version, available org URLs, and links to documentation. No authentication required beyond the base `exec:docverse` scope.

#### Superadmin — organization management

These endpoints are separated under `/admin/` to enable the `admin:docverse` Gafaelfawr scope at the ingress level.

```
POST   /admin/orgs                                  → create organization
DELETE /admin/orgs/:org                              → delete organization
```

#### Organizations

```
GET    /orgs                                         → list organizations (reader+)
GET    /orgs/:org                                    → get organization (reader+)
PATCH  /orgs/:org                                    → update org settings (admin)
```

The `GET /orgs/:org` response includes navigation URLs and the slug rewrite rules:

```json
{
  "self_url": "https://docverse.../orgs/rubin",
  "projects_url": "https://docverse.../orgs/rubin/projects",
  "members_url": "https://docverse.../orgs/rubin/members",
  "slug": "rubin",
  "title": "Rubin Observatory",
  "slug_rewrite_rules": [
    { "type": "ignore", "glob": "dependabot/**" },
    { "type": "prefix_strip", "prefix": "tickets/", "edition_kind": "draft" }
  ]
}
```

#### Slug preview

```
POST   /orgs/:org/slug-preview                       → preview slug resolution (admin)
```

Tests the org's (or a specific project's) edition slug rewrite rules against a git ref without creating any resources. See the {ref}`edition-slug-rewrite-rules` section for request/response details.

#### Org membership

```
GET    /orgs/:org/members                            → list memberships (admin)
POST   /orgs/:org/members                            → add membership (admin)
GET    /orgs/:org/members/:id                        → get membership (admin)
DELETE /orgs/:org/members/:id                        → remove membership (admin)
```

Membership `:id` uses a composite key format: `user:jdoe` or `group:g_spherex`. This is self-documenting in URLs and corresponds directly to the `principal_type:principal` pair which is unique within an org.

#### Projects

```
GET    /orgs/:org/projects                           → list projects (reader+, paginated)
POST   /orgs/:org/projects                           → create project (admin)
GET    /orgs/:org/projects/:project                  → get project (reader+)
PATCH  /orgs/:org/projects/:project                  → update project (admin)
DELETE /orgs/:org/projects/:project                  → soft-delete project (admin)
```

The `GET /orgs/:org/projects/:project` response includes navigation URLs:

```json
{
  "self_url": "https://docverse.../orgs/rubin/projects/pipelines",
  "org_url": "https://docverse.../orgs/rubin",
  "editions_url": "https://docverse.../orgs/rubin/projects/pipelines/editions",
  "builds_url": "https://docverse.../orgs/rubin/projects/pipelines/builds",
  "slug": "pipelines",
  "title": "LSST Science Pipelines",
  "doc_repo": "https://github.com/lsst/pipelines_lsst_io",
  "slug_rewrite_rules": null
}
```

The `slug_rewrite_rules` field is `null` when the project inherits the org's rules, or contains a project-specific rule list when overridden.

#### Editions

```
GET    /orgs/:org/projects/:project/editions         → list editions (reader+, paginated)
POST   /orgs/:org/projects/:project/editions         → create edition (admin)
GET    /orgs/:org/projects/:project/editions/:ed     → get edition (reader+)
PATCH  /orgs/:org/projects/:project/editions/:ed     → update edition (admin)
DELETE /orgs/:org/projects/:project/editions/:ed     → soft-delete edition (admin)
```

PATCH supports setting `build` to reassign an edition to a specific build (used for rollback or manual reassignment). This enqueues an edition update task and returns a `queue_url`.

The `POST` to create an edition accepts the following request body:

```json
{
  "slug": "usdf-dev--main",
  "title": "USDF Dev — main",
  "kind": "alternate",
  "tracking_mode": "alternate_git_ref",
  "tracking_params": {
    "git_ref": "main",
    "alternate_name": "usdf-dev"
  }
}
```

| Field             | Type   | Required | Description                                                                                         |
| ----------------- | ------ | -------- | --------------------------------------------------------------------------------------------------- |
| `slug`            | string | yes      | URL-safe edition identifier. Must be unique within the project.                                      |
| `title`           | string | yes      | Human-readable title for dashboards and metadata.                                                    |
| `kind`            | string | yes      | Edition kind (`main`, `release`, `draft`, `major`, `minor`, `alternate`). Controls dashboard grouping and lifecycle rule targeting. |
| `tracking_mode`   | string | yes      | One of the supported tracking modes (see {ref}`projects`). Determines which builds update this edition. |
| `tracking_params` | object | no       | Mode-specific parameters (e.g., `{"git_ref": "main"}` for `git_ref` mode, or `{"git_ref": "main", "alternate_name": "usdf-dev"}` for `alternate_git_ref` mode). Required for parameterized tracking modes. |

```
GET    /orgs/:org/projects/:project/editions/:ed/history → edition-build history (reader+, paginated)
```

The history endpoint returns a paginated list of builds the edition has pointed to, ordered by position (most recent first). Each entry includes the build reference, timestamp, and position.

The `GET /orgs/:org/projects/:project/editions/:ed` response:

```json
{
  "self_url": "https://docverse.../orgs/rubin/projects/pipelines/editions/__main",
  "project_url": "https://docverse.../orgs/rubin/projects/pipelines",
  "build_url": "https://docverse.../orgs/rubin/projects/pipelines/builds/01HQ-3KBR-T5GN-8W",
  "history_url": "https://docverse.../orgs/rubin/projects/pipelines/editions/__main/history",
  "published_url": "https://pipelines.lsst.io/",
  "slug": "__main",
  "kind": "main",
  "tracking_mode": "git_ref",
  "tracking_params": { "git_ref": "main" },
  "date_updated": "2026-02-08T12:00:00Z"
}
```

#### Builds

```
GET    /orgs/:org/projects/:project/builds           → list builds (reader+, paginated)
POST   /orgs/:org/projects/:project/builds           → create build (uploader+)
GET    /orgs/:org/projects/:project/builds/:build    → get build (reader+)
PATCH  /orgs/:org/projects/:project/builds/:build    → signal upload complete (uploader+)
DELETE /orgs/:org/projects/:project/builds/:build    → soft-delete build (admin)
```

The `POST` to create a build returns a single presigned upload URL for the tarball:

Request:

```json
{
  "git_ref": "main",
  "alternate_name": "usdf-dev",
  "content_hash": "sha256:a1b2c3d4...",
  "annotations": { "kind": "release" }
}
```

The `alternate_name` field is optional — most projects don't use it. When present, it scopes the build to alternate-aware editions. Builds with `alternate_name` are **not** matched by generic `git_ref`-only tracking editions — the alternate name acts as a namespace. See {ref}`alternate-scoped-editions` in the projects section for details on how alternate names interact with edition tracking and slug derivation.

Response:

```json
{
  "self_url": "https://docverse.../orgs/rubin/projects/pipelines/builds/01HQ-3KBR-T5GN-8W",
  "project_url": "https://docverse.../orgs/rubin/projects/pipelines",
  "id": "01HQ-3KBR-T5GN-8W",
  "status": "uploading",
  "git_ref": "main",
  "upload_url": "https://storage.googleapis.com/docverse-staging-rubin/__staging/01HQ...tar.gz?sig=...",
  "date_created": "2026-02-08T12:00:00Z"
}
```

The `upload_url` is a presigned PUT URL for the tarball. It points to either the staging bucket (if configured) or the publishing bucket's `__staging/` prefix. The client uploads the tarball via HTTP PUT to this URL, then signals upload complete via PATCH.

The `content_hash` field allows the server to verify tarball integrity after upload. The `files` field from the per-file upload model is removed -- the file manifest is discovered during the unpack step.

The `PATCH` to signal upload complete updates the build status and enqueues background processing:

Request:

```json
{
  "status": "uploaded"
}
```

Response:

```json
{
  "self_url": "https://docverse.../orgs/rubin/projects/pipelines/builds/01HQ-3KBR-T5GN-8W",
  "queue_url": "https://docverse.../queue/jobs/0G4R-MFBZ-K7QP-5X",
  "status": "processing"
}
```

#### Queue jobs

```
GET    /queue/jobs/:job                              → get job status (authenticated)
```

Queue jobs provide status tracking for background operations (build processing, edition updates, dashboard rendering). The job resource is identified by a Docverse-owned Crockford Base32 ID. See the {ref}`queue` section for the `QueueJob` table schema and progress tracking implementation.

```json
{
  "self_url": "https://docverse.../queue/jobs/0G4R-MFBZ-K7QP-5X",
  "id": "0G4R-MFBZ-K7QP-5X",
  "status": "in_progress",
  "kind": "build_processing",
  "build_url": "https://docverse.../orgs/rubin/projects/pipelines/builds/01HQ-3KBR-T5GN-8W",
  "date_created": "2026-02-08T12:00:00Z",
  "date_started": "2026-02-08T12:00:01Z",
  "date_completed": null,
  "phase": "editions",
  "progress": {
    "editions_total": 3,
    "editions_completed": [
      { "slug": "__main", "published_url": "https://pipelines.lsst.io/" }
    ],
    "editions_failed": [],
    "editions_in_progress": ["v2.x", "DM-12345"]
  }
}
```
