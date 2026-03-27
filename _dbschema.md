(dbschema)=

## Database models

Docverse stores all state in a PostgreSQL database accessed through SQLAlchemy (async, via Safir's database utilities).
This section provides a centralized reference for the database schema.
Individual sections describe the behavioral design around each table; this section focuses on column definitions and relationships.

### Entity-relationship diagram

```{mermaid}
erDiagram
    Organization ||--o{ Project : "has"
    Organization ||--o{ OrgMembership : "has"
    Organization ||--o{ organization_credentials : "has"
    Organization ||--o{ organization_services : "has"
    organization_services }o--|| organization_credentials : "uses"
    Organization ||--o{ DashboardTemplate : "has"
    Organization ||--o{ QueueJob : "scoped to"
    Project ||--o{ Build : "has"
    Project ||--o{ Edition : "has"
    Project ||--o{ QueueJob : "scoped to"
    Project ||--o{ DashboardTemplate : "overrides"
    Build ||--o{ BuildObject : "inventories"
    Build ||--o{ EditionBuildHistory : "logged in"
    Build ||--o{ QueueJob : "tracked by"
    Edition ||--o{ EditionBuildHistory : "logged in"
    Edition }o--o| Build : "current build"

    Organization {
        int id PK
        string slug UK
        string title
        string base_domain
        enum url_scheme
        string root_path_prefix
        JSONB slug_rewrite_rules
        JSONB lifecycle_rules
        string publishing_store_label
        string staging_store_label
        string cdn_service_label
        string dns_service_label
        datetime date_created
        datetime date_updated
    }

    Project {
        int id PK
        string slug
        string title
        int org_id FK
        string doc_repo
        JSONB slug_rewrite_rules
        JSONB lifecycle_rules
        datetime date_created
        datetime date_updated
        datetime date_deleted
    }

    Build {
        int id PK
        string public_id
        int project_id FK
        string git_ref
        string alternate_name
        string content_hash
        enum status
        string staging_key
        int object_count
        bigint total_size_bytes
        string uploader
        JSONB annotations
        datetime date_created
        datetime date_uploaded
        datetime date_completed
        datetime date_deleted
    }

    Edition {
        int id PK
        string slug
        string title
        int project_id FK
        enum kind
        enum tracking_mode
        JSONB tracking_params
        int current_build_id FK
        bool lifecycle_exempt
        datetime date_created
        datetime date_updated
        datetime date_deleted
    }

    OrgMembership {
        int id PK
        int org_id FK
        string principal
        enum principal_type
        enum role
    }

    organization_credentials {
        int id PK
        int organization_id FK
        string label
        string provider
        bytea encrypted_credentials
        datetime date_created
        datetime date_updated
    }

    organization_services {
        int id PK
        int organization_id FK
        string label
        string category
        string provider
        JSONB config
        string credential_label
        datetime date_created
        datetime date_updated
    }

    BuildObject {
        int id PK
        int build_id FK
        string key
        string content_hash
        string content_type
        bigint size
    }

    EditionBuildHistory {
        int id PK
        int edition_id FK
        int build_id FK
        int position
        datetime date_created
    }

    DashboardTemplate {
        int id PK
        int org_id FK
        int project_id FK
        string github_owner
        string github_repo
        string path
        string git_ref
        string store_prefix
        string sync_id
        datetime date_synced
    }

    QueueJob {
        int id PK
        int public_id
        string backend_job_id
        enum kind
        enum status
        string phase
        int org_id FK
        int project_id FK
        int build_id FK
        JSONB progress
        JSONB errors
        datetime date_created
        datetime date_started
        datetime date_completed
    }
```

### Core domain tables

These tables define the primary domain model for Docverse.

(table-organization)=

#### Organization

The organization is the top-level resource and the sole infrastructure configuration boundary.
All projects within an org share the same object store, CDN, root domain, URL scheme, and default dashboard templates.
See {ref}`organizations` for the full behavioral design.

| Column                  | Type              | Description |
| ----------------------- | ----------------- | ----------- |
| `id`                    | int               | Primary key |
| `slug`                  | str (unique)      | URL-safe identifier (e.g., `rubin`, `spherex`) |
| `title`                 | str               | Human-readable name |
| `base_domain`           | str               | Root domain for published URLs (e.g., `lsst.io`) |
| `url_scheme`            | enum              | `subdomain` or `path_prefix` — determines how project URLs are constructed |
| `root_path_prefix`      | str               | Path prefix for path-prefix URL scheme (e.g., `/documentation/`) |
| `slug_rewrite_rules`    | JSONB             | Ordered list of edition slug rewrite rules (see {ref}`edition-slug-rewrite-rules`) |
| `lifecycle_rules`       | JSONB             | Default lifecycle rules for projects in this org (see {ref}`projects`) |
| `publishing_store_label` | str (nullable)   | Label of the `object_storage` service used as the publishing store |
| `staging_store_label`   | str (nullable)    | Label of the `object_storage` service used for staging uploads |
| `cdn_service_label`     | str (nullable)    | Label of the `cdn` service used for cache purging and edge data |
| `dns_service_label`     | str (nullable)    | Label of the `dns` service used for subdomain registration |
| `date_created`          | datetime          | Creation timestamp |
| `date_updated`          | datetime          | Last modification timestamp |

Infrastructure connections are configured through the three-layer model described in {ref}`org-infrastructure`: credentials ({ref}`table-organization-credentials`), services ({ref}`table-organization-services`), and slot assignments (the `_label` columns above).

(table-project)=

#### Project

A documentation site with a stable URL and multiple versions (editions).
Projects belong to an organization and inherit its infrastructure and default configuration, with optional per-project overrides for slug rewrite rules and lifecycle rules.
See {ref}`projects` for the full behavioral design.

| Column               | Type                    | Description |
| -------------------- | ----------------------- | ----------- |
| `id`                 | int                     | Primary key |
| `slug`               | str                     | URL-safe identifier, unique within org (e.g., `sqr-006`, `pipelines`) |
| `title`              | str                     | Human-readable name |
| `org_id`             | FK → Organization       | Owning organization |
| `doc_repo`           | str                     | GitHub repository URL for the documentation source |
| `slug_rewrite_rules` | JSONB (nullable)        | When set, completely replaces the org-level rules for this project |
| `lifecycle_rules`    | JSONB (nullable)        | When set, overrides the org-level lifecycle rules for this project |
| `date_created`       | datetime                | Creation timestamp |
| `date_updated`       | datetime                | Last modification timestamp |
| `date_deleted`       | datetime (nullable)     | Soft-delete timestamp; `null` when active |

The `projects` table requires the `pg_trgm` extension and has GIN trigram indexes on the `slug` and `title` columns to support the fuzzy search endpoint (`q` parameter on `GET /orgs/:org/projects`).

(table-build)=

#### Build

A discrete upload of documentation content for a project.
Builds are conceptually immutable after processing and carry metadata about their origin.
Builds are identified externally with a Crockford Base32 ID.
See {ref}`projects` for the upload flow and processing pipeline.

| Column             | Type                    | Description |
| ------------------ | ----------------------- | ----------- |
| `id`               | int                     | Internal primary key |
| `public_id`        | Crockford Base32        | Externally-visible identifier (e.g., `01HQ-3KBR-T5GN-8W`) |
| `project_id`       | FK → Project            | Owning project |
| `git_ref`          | str                     | Git branch or tag that produced this build |
| `alternate_name`   | str (nullable)          | Deployment/variant scope (e.g., `usdf-dev`); see {ref}`alternate-scoped-editions` |
| `content_hash`     | str                     | SHA-256 hash of the uploaded tarball for integrity verification |
| `status`           | enum                    | `pending`, `uploaded`, `processing`, `completed`, `failed` (`uploaded` is a signal-only value — never stored) |
| `staging_key`      | str                     | Object store key for the uploaded tarball (e.g., `__staging/{build_id}.tar.gz`) |
| `object_count`     | int                     | Number of files extracted from the tarball (populated during inventory) |
| `total_size_bytes` | bigint                  | Total size of extracted content (populated during inventory) |
| `uploader`         | str                     | Username of the authenticated uploader |
| `annotations`      | JSONB                   | Client-provided metadata about the build |
| `date_created`     | datetime                | When the build record was created |
| `date_uploaded`    | datetime (nullable)     | When the client signaled upload complete |
| `date_completed`   | datetime (nullable)     | When background processing finished |
| `date_deleted`     | datetime (nullable)     | Soft-delete timestamp; `null` when active |

(table-edition)=

#### Edition

A named, published view of a project's documentation at a stable URL.
Editions are pointers — they represent a specific build's content served at an edition-specific URL path (e.g., `/v/main/`, `/v/DM-12345/`).
See {ref}`projects` for tracking modes, edition kinds, and auto-creation behavior.

| Column             | Type                    | Description |
| ------------------ | ----------------------- | ----------- |
| `id`               | int                     | Primary key |
| `slug`             | str                     | URL-safe identifier, unique within project (e.g., `__main`, `DM-12345`, `v2.x`) |
| `title`            | str                     | Human-readable name for dashboards |
| `project_id`       | FK → Project            | Owning project |
| `kind`             | enum                    | `main`, `release`, `draft`, `major`, `minor`, `alternate` — classifies for dashboards and lifecycle rules |
| `tracking_mode`    | enum                    | Determines which builds update this edition (see {ref}`projects` for the full list) |
| `tracking_params`  | JSONB                   | Mode-specific parameters (e.g., `{"git_ref": "main"}`, `{"major_version": 2}`) |
| `current_build_id` | FK → Build (nullable)   | The build currently served at this edition's URL; `null` before first build |
| `lifecycle_exempt`  | bool                   | When `true`, this edition is never deleted by lifecycle rules |
| `date_created`     | datetime                | Creation timestamp |
| `date_updated`     | datetime                | Last update timestamp (changes when build pointer moves) |
| `date_deleted`     | datetime (nullable)     | Soft-delete timestamp; `null` when active |

### Supporting tables

These tables support the core domain model with membership, credentials, object inventory, history tracking, dashboard templates, and background job management.
Each is discussed in detail in its respective section.

(table-org-membership)=

#### OrgMembership

Maps users and groups to roles within organizations.
See {ref}`auth` for the full authorization model, role definitions, and resolution algorithm.

| Column           | Type              | Description |
| ---------------- | ----------------- | ----------- |
| `id`             | int               | Auto-increment primary key |
| `org_id`         | FK → Organization | The organization |
| `principal`      | str               | A username or group name |
| `principal_type` | enum              | `user` or `group` |
| `role`           | enum              | `reader`, `uploader`, or `admin` |

(table-organization-credentials)=

#### organization_credentials

Stores Fernet-encrypted authentication secrets for cloud providers.
Each credential represents a single provider's authentication (e.g., one AWS access key pair, one Cloudflare API token).
A credential can be shared across multiple services from the same provider.
See {ref}`organizations` for the encryption scheme, key rotation, and the three-layer infrastructure model.

| Column                  | Type              | Description |
| ----------------------- | ----------------- | ----------- |
| `id`                    | int               | Auto-increment primary key |
| `organization_id`       | FK → Organization | Owning organization |
| `label`                 | str               | Human-friendly name (e.g., `cloudflare`, `aws-prod`) |
| `provider`              | str               | Cloud provider identifier: `aws`, `cloudflare`, `fastly`, `gcp`, or `s3` |
| `encrypted_credentials` | bytea             | Fernet token containing the encrypted credential payload (JSON-encoded) |
| `date_created`          | datetime          | Creation timestamp |
| `date_updated`          | datetime          | Last modification timestamp |

Unique constraint on `(organization_id, label)`.

(table-organization-services)=

#### organization_services

Stores non-secret infrastructure configuration for organization services.
Each service pairs a configuration (bucket name, account ID, etc.) with a reference to a credential from {ref}`table-organization-credentials`.
The `config` column is plaintext JSONB and is returned in API responses, unlike credentials which are write-only.
See {ref}`org-infrastructure` for the full service provider catalog and examples.

| Column             | Type              | Description |
| ------------------ | ----------------- | ----------- |
| `id`               | int               | Auto-increment primary key |
| `organization_id`  | FK → Organization | Owning organization |
| `label`            | str               | Human-friendly name (e.g., `docs-bucket`, `cdn`, `dns`) |
| `category`         | str               | Service category: `object_storage`, `cdn`, or `dns` |
| `provider`         | str               | Service provider (e.g., `aws_s3`, `cloudflare_r2`, `fastly`, `cloudflare_workers`, `route53`, `cloudflare_dns`) |
| `config`           | JSONB             | Provider-specific non-secret configuration (bucket name, account ID, zone ID, etc.) |
| `credential_label` | str               | Label of the credential in `organization_credentials` that this service uses for authentication |
| `date_created`     | datetime          | Creation timestamp |
| `date_updated`     | datetime          | Last modification timestamp |

Unique constraint on `(organization_id, label)`.
The `category` is derivable from `provider` but stored explicitly for query convenience (e.g., "list all object_storage services for this org").

(table-build-object)=

#### BuildObject

Inventories every file extracted from a build's tarball.
Populated during the inventory phase of build processing.
See {ref}`projects` for how the inventory enables diff-based edition updates and orphan detection.

| Column         | Type         | Description |
| -------------- | ------------ | ----------- |
| `id`           | int          | Primary key |
| `build_id`     | FK → Build   | Owning build |
| `key`          | str          | Object store path (e.g., `__builds/{build_id}/index.html`) |
| `content_hash` | str          | ETag or SHA-256 hash of the object content |
| `content_type` | str          | MIME type (e.g., `text/html`, `image/png`) |
| `size`         | bigint       | Object size in bytes |

(table-edition-build-history)=

#### EditionBuildHistory

Logs every build that an edition has pointed to, enabling rollback and orphan detection.
See {ref}`projects` for the rollback API and how lifecycle rules reference history position.

| Column       | Type           | Description |
| ------------ | -------------- | ----------- |
| `id`         | int            | Primary key |
| `edition_id` | FK → Edition   | The edition |
| `build_id`   | FK → Build     | The build that was served |
| `position`   | int            | Ordering position (1 = most recent) |
| `date_created` | datetime     | When this history entry was recorded |

(table-dashboard-template)=

#### DashboardTemplate

Tracks dashboard template sources (GitHub repos) and their sync state.
See {ref}`dashboards` for the template directory structure, sync flow, and rendering pipeline.

| Column         | Type                    | Description |
| -------------- | ----------------------- | ----------- |
| `id`           | int                     | Primary key |
| `org_id`       | FK → Organization       | Owning organization |
| `project_id`   | FK → Project (nullable) | If set, this is a project-level override |
| `github_owner` | str                     | GitHub organization or user |
| `github_repo`  | str                     | Repository name |
| `path`         | str                     | Path within repo (default `""` for root) |
| `git_ref`      | str                     | Branch or tag to track |
| `store_prefix` | str (nullable)          | Object store prefix for current synced template files |
| `sync_id`      | str (nullable)          | Current sync version identifier (timestamp-based) |
| `date_synced`  | datetime (nullable)     | Last successful sync timestamp |

Unique constraint on `(org_id, project_id)` — at most one template per org (where `project_id` is null) and one per project.

(table-queue-job)=

#### QueueJob

Tracks all background jobs as the single source of truth for job state and progress.
The queue backend (Arq/Redis) handles delivery; this table is the authoritative state store.
See {ref}`queue` for progress tracking, cross-job serialization, and operator queries.

| Column           | Type                    | Description |
| ---------------- | ----------------------- | ----------- |
| `id`             | int                     | Internal primary key |
| `public_id`      | int                     | Crockford Base32 serialized in API |
| `backend_job_id` | str (nullable)          | Reference to the queue backend's job ID (e.g., Arq UUID) |
| `kind`           | enum                    | `build_processing`, `edition_update`, `dashboard_sync`, `lifecycle_eval`, `git_ref_audit`, `purgatory_cleanup`, `credential_reencrypt` |
| `status`         | enum                    | `queued`, `in_progress`, `completed`, `completed_with_errors`, `failed`, `cancelled` |
| `phase`          | str (nullable)          | Current processing phase (e.g., `inventory`, `tracking`, `editions`, `dashboard`) |
| `org_id`         | FK → Organization       | Scoped to org for filtering |
| `project_id`     | FK → Project (nullable) | Set for build/edition jobs |
| `build_id`       | FK → Build (nullable)   | Set for build processing jobs |
| `progress`       | JSONB (nullable)        | Structured progress data, phase-specific |
| `errors`         | JSONB (nullable)        | Collected error details |
| `date_created`   | datetime                | When the job was enqueued |
| `date_started`   | datetime (nullable)     | When a worker picked it up |
| `date_completed` | datetime (nullable)     | When the job finished |
