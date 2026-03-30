(api)=

## REST API design

### Conventions

The Docverse REST API follows SQuaRE's REST API design conventions, and specifically follows patterns established in [Ook](https://github.com/lsst-sqre/ook) and [Times Square](https://github.com/lsst-sqre/times-square):

- **No URL versioning**: the API has a single set of paths (no `/v1/` prefix). Breaking changes would be handled by introducing new endpoints alongside deprecated ones. A lot of design work would have to go into supporting multiple API versions simultaneously, so assuming path-based API versioning seems presumptive at this stage.
- **HATEOAS-style navigation**: all resource representations include `self_url` and relevant navigation URLs (e.g., `project_url`, `org_url`, `builds_url`, `editions_url`). Clients navigate the API via these provided URLs rather than constructing their own.
- **Collections are top-level arrays**: collection endpoints return a JSON array of resource objects. Pagination metadata is carried in response headers, not a wrapper object.
- **Keyset pagination**: all collection endpoints use keyset pagination via [Safir's pagination library](https://safir.lsst.io/user-guide/database/pagination.html). Common query parameters:
  - `cursor` — opaque pagination cursor string (omit for the first page).
  - `limit` — page size, between 1 and 100 (default 25).
  - `order` — sort field (endpoint-specific; see individual endpoint docs).
  Response headers:
  - `Link` — [RFC 8288](https://www.rfc-editor.org/rfc/rfc8288) links with `rel="first"`, `rel="next"`, and `rel="prev"` as applicable.
  - `X-Total-Count` — total number of matching entries (before pagination).
  Collections remain top-level JSON arrays; pagination metadata is carried only in headers.
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

A single Gafaelfawr ingress protects the API:

| Ingress path | Required scope  | Purpose                    |
| ------------ | --------------- | -------------------------- |
| `/*`         | `exec:docverse` | All Docverse API routes    |

`/admin/*` routes are accessible at the ingress level but restricted to superadmin users (determined by `superadmin_usernames` configuration) at the application layer. All org-level authorization (admin vs uploader vs reader) is also enforced at the **application layer** via `OrgMembership` checks. Gafaelfawr ingresses cannot express "user X has role Y in org Z" — that granularity requires application-level logic. The ingress layer ensures the user is authenticated and has basic Docverse access; the application layer checks the specific role required for each endpoint.

### Endpoint catalog

#### Root

```
GET  /                                              → API metadata and navigation URLs
```

Returns API version, available org URLs, and links to documentation. No authentication required beyond the base `exec:docverse` scope.

#### Superadmin — organization management

These endpoints are separated under `/admin/` for organizational clarity. Access is restricted to superadmin users (determined by `superadmin_usernames` configuration) at the application layer.

```
GET    /admin/orgs                                   → list all organizations
POST   /admin/orgs                                   → create organization
GET    /admin/orgs/:org                              → get organization
DELETE /admin/orgs/:org                              → delete organization
```

The `POST /admin/orgs` request body accepts an optional `members` array of `OrgMembershipCreate` objects, allowing the superadmin to seed initial org membership in a single request.

Admin organization responses include an `org_url` field linking to the org-scoped `GET /orgs/:org` endpoint, enabling HATEOAS cross-navigation between admin and org-scoped views.

#### Organizations

```
GET    /orgs                                         → list organizations (reader+)
GET    /orgs/:org                                    → get organization (reader+)
PATCH  /orgs/:org                                    → update org settings (admin)
```

The `GET /orgs/:org` response includes navigation URLs, infrastructure service slot assignments as embedded summaries, and the slug rewrite rules:

```json
{
  "self_url": "https://docverse.../orgs/rubin",
  "services_url": "https://docverse.../orgs/rubin/services",
  "credentials_url": "https://docverse.../orgs/rubin/credentials",
  "projects_url": "https://docverse.../orgs/rubin/projects",
  "members_url": "https://docverse.../orgs/rubin/members",
  "slug": "rubin",
  "title": "Rubin Observatory",
  "base_domain": "lsst.io",
  "url_scheme": "subdomain",
  "publishing_store": {
    "self_url": "https://docverse.../orgs/rubin/services/docs-bucket",
    "label": "docs-bucket",
    "category": "object_storage",
    "provider": "cloudflare_r2"
  },
  "staging_store": {
    "self_url": "https://docverse.../orgs/rubin/services/staging-bucket",
    "label": "staging-bucket",
    "category": "object_storage",
    "provider": "cloudflare_r2"
  },
  "cdn_service": {
    "self_url": "https://docverse.../orgs/rubin/services/cdn",
    "label": "cdn",
    "category": "cdn",
    "provider": "cloudflare_workers"
  },
  "dns_service": null,
  "slug_rewrite_rules": [
    { "type": "ignore", "glob": "dependabot/**" },
    { "type": "prefix_strip", "prefix": "tickets/", "edition_kind": "draft" }
  ],
  "default_edition_config": {
    "tracking_mode": "git_ref",
    "tracking_params": { "git_ref": "main" },
    "title": "Main",
    "lifecycle_exempt": true
  }
}
```

Service slots use embedded summaries (label, category, provider, and a HATEOAS URL to the full service detail) rather than bare label strings.
Write operations (`POST`, `PATCH`) use `_label` suffixed fields (e.g., `publishing_store_label`) to assign services to slots.
See {ref}`org-infrastructure` for the full infrastructure model.

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

#### Credentials

```
GET    /orgs/:org/credentials                          → list credentials (admin)
POST   /orgs/:org/credentials                          → create credential (admin)
GET    /orgs/:org/credentials/:label                   → get credential metadata (admin)
DELETE /orgs/:org/credentials/:label                   → delete credential (admin)
```

Credentials store provider-level authentication secrets (API tokens, access keys).
The `POST` request includes `label` and a `credentials` object whose schema depends on the `provider` field (see {ref}`org-infrastructure` for per-provider schemas).
`GET` responses return metadata (label, provider, timestamps) but **never** the decrypted secrets.
A credential cannot be deleted while any service references it.

#### Services

```
GET    /orgs/:org/services                             → list services (admin)
POST   /orgs/:org/services                             → create service (admin)
GET    /orgs/:org/services/:label                      → get service detail (admin)
PATCH  /orgs/:org/services/:label                      → update service (admin)
DELETE /orgs/:org/services/:label                      → delete service (admin)
```

Services combine non-secret infrastructure configuration with a credential reference.
The `POST` request includes `label`, a `config` object (whose schema depends on the `provider` field), and `credential_label` (referencing an existing credential).
At creation time, Docverse validates that the credential exists and that its `provider` is compatible with the service's provider (see {ref}`org-infrastructure` for the compatibility matrix).
`GET` responses include the full non-secret `config` (bucket name, account ID, etc.).
`PATCH` accepts a partial update body.
Currently supports `credential_label` to reassign the service to a different credential.
Docverse validates that the new credential exists and that its provider is compatible with the service's provider.
A service cannot be deleted while any organization slot references it.

The `GET /orgs/:org/services/:label` response:

```json
{
  "self_url": "https://docverse.../orgs/rubin/services/docs-bucket",
  "org_url": "https://docverse.../orgs/rubin",
  "credential_url": "https://docverse.../orgs/rubin/credentials/cloudflare",
  "label": "docs-bucket",
  "category": "object_storage",
  "provider": "cloudflare_r2",
  "config": {
    "account_id": "abc123",
    "bucket": "rubin-docs"
  },
  "credential_label": "cloudflare",
  "date_created": "2026-01-15T00:00:00Z",
  "date_updated": "2026-03-19T00:00:00Z"
}
```

#### Projects

```
GET    /orgs/:org/projects                           → list projects (reader+, paginated)
POST   /orgs/:org/projects                           → create project (admin)
GET    /orgs/:org/projects/:project                  → get project (reader+)
PATCH  /orgs/:org/projects/:project                  → update project (admin)
DELETE /orgs/:org/projects/:project                  → soft-delete project (admin)
```

Query parameters for `GET /orgs/:org/projects`:

| Parameter | Type   | Default | Description                                           |
| --------- | ------ | ------- | ----------------------------------------------------- |
| `order`   | string | `slug`  | Sort field: `slug` (ASC) or `date_created` (DESC)     |
| `cursor`  | string | —       | Opaque pagination cursor                               |
| `limit`   | int    | 25      | Page size (1–100)                                      |
| `q`       | string | —       | Fuzzy search query (1–256 characters)                  |

The `q` parameter enables fuzzy search across project slug and title using PostgreSQL trigram similarity (`pg_trgm`). When `q` is provided, results are ranked by relevance (highest similarity score first) rather than the `order` field, and the `cursor` parameter cannot be used — combining `q` with `cursor` returns a 422 error. A minimum similarity threshold of 0.1 filters low-quality matches. Search results are returned as a single page (up to `limit` entries) with no pagination cursors.

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

Query parameters for `GET .../editions`:

| Parameter | Type   | Default | Description                                                           |
| --------- | ------ | ------- | --------------------------------------------------------------------- |
| `order`   | string | `slug`  | Sort field: `slug` (ASC), `date_created` (DESC), or `date_updated` (DESC) |
| `kind`    | string | —       | Filter by edition kind (`main`, `release`, `draft`, etc.)             |
| `cursor`  | string | —       | Opaque pagination cursor                                               |
| `limit`   | int    | 25      | Page size (1–100)                                                      |

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

Query parameters for `GET .../builds`:

| Parameter | Type   | Default | Description                                                    |
| --------- | ------ | ------- | -------------------------------------------------------------- |
| `status`  | string | —       | Filter by build status (`pending`, `processing`, `completed`, `failed`) |
| `cursor`  | string | —       | Opaque pagination cursor                                        |
| `limit`   | int    | 25      | Page size (1–100)                                               |

Builds are always sorted by `date_created` descending (newest first).

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
  "status": "pending",
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
