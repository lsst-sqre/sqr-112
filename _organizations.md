(organizations)=

## Organizations design

With _organizations_, a single Docverse deployment can host multiple documentation domains.
Doing so can enable a single institution (like AURA or NOIRLab) to host documentation for multiple multiple missions with completely separate white-labelled presentations.
It can also enable SQuaRE to provide Docverse-as-a-service for partner institutions.

The Organization is the sole infrastructure configuration boundary — all projects within an org share the same object store, CDN, root domain, URL scheme, and default dashboard templates.

Docverse follows a "bring your own infrastructure" strategy: each organization provisions and owns its cloud resources (object store buckets, CDN services, DNS zones) rather than Docverse providing centralized, shared infrastructure.
This design is motivated by three concerns.
First, **cost allocation** — organizations pay for and control their own cloud spend directly, avoiding the need for Docverse to meter usage or redistribute costs.
Second, **data ownership** — organizations retain full ownership of their stored documentation artifacts; Docverse itself only stores connection metadata and encrypted credentials, never the data at rest.
Third, **regulatory compliance** — some organizations face restrictions such as ITAR export controls that dictate where documentation can be hosted and who can access the underlying storage, requirements that are simplest to satisfy when the organization controls its own accounts.

The {ref}`org-infrastructure` section below describes how organizations configure credentials, services, and slot assignments to connect their cloud resources to Docverse.

### Organization configuration

Each organization owns:

- **Infrastructure services** (credentials, services, and slot assignments — see {ref}`org-infrastructure` below):
  - _Publishing store_: object store bucket where published documentation is served from.
  - _Staging store_ (optional): separate object store bucket for build tarball uploads. When configured, presigned upload URLs point to the staging bucket and the Docverse worker reads tarballs from it. When not configured, staging uses a `__staging/` prefix in the publishing bucket. The staging store optimization is useful when the publishing store is on a different network than the Docverse compute cluster — for example, a GCS staging bucket in the same region as a GKE cluster paired with a Cloudflare R2 publishing bucket. The staging bucket only needs transient storage; tarballs are deleted after processing.
  - _CDN_ (optional): cache purging and edge data store updates. The CDN is provisioned externally; Docverse only interacts at runtime. Organizations that self-host or use a custom serving layer leave this slot empty.
  - _DNS_ (optional): for subdomain-based layouts, Docverse registers subdomains via DNS APIs. When using Cloudflare, wildcard subdomains are supported on all plans via a proxied `*.domain` DNS record with free wildcard SSL. Organizations that manage DNS externally leave this slot empty.
- **URL scheme** (per-org setting, one of):
  - _Subdomain_: each project gets `project.base-domain` (e.g., `sqr-006.lsst.io`)
  - _Path-prefix_: all projects under a root path (e.g., `example.com/documentation/project`)
- **Base domain** (e.g., `lsst.io`) and **root path prefix** (for path-prefix mode).
- **Dashboard templates**: a GitHub repo containing Jinja templates and assets, configured at the org level with optional per-project overrides. See the [Dashboard Templating System section](#dashboards) for full details.
- **Edition slug rewrite rules**: an ordered list of rules that transform git refs into edition slugs. Configured at the org level with optional per-project overrides. See the {ref}`edition-slug-rewrite-rules` section for the full rule format.
- **Default edition lifecycle rules** ({ref}`projects`).
- **Default edition configuration** (`default_edition_config`): a JSONB object that sets default tracking behavior for the `__main` edition auto-created on every new project. When a project creation request omits its own default edition configuration, the organization's `default_edition_config` is used. See {ref}`default-edition-config` in the projects section for the full precedence chain and model definition.

(org-infrastructure)=

### Infrastructure service model

Docverse follows a "bring your own infrastructure" strategy: each organization provisions and owns its cloud resources (object store buckets, CDN services, DNS zones).
Docverse stores only connection metadata and encrypted credentials, never the documentation data itself.

Infrastructure configuration uses a three-layer model that separates authentication from service configuration and role assignment:

1. **Credentials** — provider-level authentication (e.g., one AWS credential, one Cloudflare API token). A single credential can be shared across multiple services from the same provider.
2. **Services** — infrastructure configurations that pair non-secret config (bucket name, account ID, zone ID) with a reference to a credential. Each service has a category (`object_storage`, `cdn`, or `dns`) and a provider-specific configuration schema.
3. **Slot assignments** — the organization references services by label for specific roles: `publishing_store`, `staging_store`, `cdn_service`, and `dns_service`.

This separation provides several benefits:
- **Credential deduplication**: one Cloudflare API token serves R2, Workers, and DNS services simultaneously.
- **Rotate once**: updating a credential automatically applies to all services that reference it.
- **Visible configuration**: GET responses on services show non-secret config (bucket name, account ID, region) while secrets remain encrypted and write-only.
- **Flexible composition**: organizations can mix providers (e.g., AWS S3 + Fastly CDN) or go all-in on one (e.g., Cloudflare for everything). Organizations that manage CDN or DNS externally simply leave those slots unassigned.

#### Credential providers

Each credential stores the authentication secrets for a single cloud provider.
The credential's `provider` field determines the schema of the encrypted payload:

| Provider      | Encrypted fields                        | Used by service providers                     |
| ------------- | --------------------------------------- | --------------------------------------------- |
| `aws`         | `access_key_id`, `secret_access_key`    | `aws_s3`, `cloudfront`, `route53`             |
| `cloudflare`  | `api_token`                             | `cloudflare_workers`, `cloudflare_dns`        |
| `fastly`      | `api_token`                             | `fastly`                                      |
| `gcp`         | `service_account_json`                  | `gcs`, `google_cloud_cdn`                     |
| `s3`          | `access_key_id`, `secret_access_key`    | `cloudflare_r2`, `minio`                      |

The `s3` credential provider uses the same field names as `aws` (`access_key_id` and `secret_access_key`) but represents generic S3-compatible credentials that are not tied to AWS IAM.
Cloudflare R2 and MinIO both expose S3-compatible APIs using their own access key systems, which are distinct from AWS credentials.
The `aws` credential is reserved for services that use AWS-native IAM credentials.

Credentials are created via `POST /orgs/:org/credentials` and are **write-only** — the GET response returns metadata (label, provider, timestamps) but never the decrypted secrets.

#### Service providers

Each service combines non-secret configuration with a reference to a credential.
The service's `provider` field determines the config schema and which credential providers are compatible:

**Object storage providers:**

| Provider         | Config fields                  | Compatible credential | Notes |
| ---------------- | ------------------------------ | --------------------- | ----- |
| `aws_s3`         | `bucket`, `region`             | `aws`                 | Native AWS S3 |
| `cloudflare_r2`  | `account_id`, `bucket`         | `s3`                  | S3-compatible API, zero egress |
| `minio`          | `endpoint_url`, `bucket`       | `s3`                  | S3-compatible API |
| `gcs`            | `bucket`, `project_id`         | `gcp`                 | Google Cloud Storage |

**CDN providers:**

| Provider              | Config fields               | Compatible credential | Notes |
| --------------------- | --------------------------- | --------------------- | ----- |
| `fastly`              | `service_id`                | `fastly`              | Surrogate-key purge, KV Store |
| `cloudflare_workers`  | `account_id`, `zone_id`     | `cloudflare`          | Edge compute + Workers KV |
| `cloudfront`          | `distribution_id`           | `aws`                 | Lambda@Edge, no pointer mode |
| `google_cloud_cdn`    | `backend_service`           | `gcp`                 | Copy mode only |

**DNS providers:**

| Provider         | Config fields       | Compatible credential |
| ---------------- | ------------------- | --------------------- |
| `route53`        | `hosted_zone_id`    | `aws`                 |
| `cloudflare_dns` | `zone_id`           | `cloudflare`          |

Services are created via `POST /orgs/:org/services` and managed at `GET/DELETE /orgs/:org/services/:label`.
Unlike credentials, the non-secret `config` fields **are** returned in GET responses, so administrators can verify bucket names, account IDs, and other infrastructure details without needing to inspect the cloud provider directly.

#### Slot assignments

The organization model has four service slots that reference services by label:

| Slot                | Required category | Purpose |
| ------------------- | ----------------- | ------- |
| `publishing_store`  | `object_storage`  | Bucket where published documentation is served from |
| `staging_store`     | `object_storage`  | Bucket for build tarball uploads (optional) |
| `cdn_service`       | `cdn`             | Cache purging and edge data store updates (optional) |
| `dns_service`       | `dns`             | Subdomain registration (optional) |

Slots are assigned via `PATCH /orgs/:org` using `_label` suffixed fields (e.g., `publishing_store_label`).
In GET responses, slots are returned as embedded service summaries (label, category, provider, and a HATEOAS URL to the full service detail) rather than bare labels.

#### Example: Cloudflare R2 + Workers setup

R2 storage uses S3-compatible credentials (separate from the Cloudflare API token used for Workers and DNS), so this setup requires two credentials:

```json
// 1. Create an S3-compatible credential for R2 storage
// POST /orgs/rubin/credentials
{
    "label": "r2",
    "credentials": {
        "provider": "s3",
        "access_key_id": "r2-access-key-xxx",
        "secret_access_key": "r2-secret-key-xxx"
    }
}

// Create a Cloudflare credential for Workers and DNS
// POST /orgs/rubin/credentials
{
    "label": "cloudflare",
    "credentials": {
        "provider": "cloudflare",
        "api_token": "cf-token-xxx"
    }
}

// 2. Create services referencing appropriate credentials
// POST /orgs/rubin/services
{
    "label": "docs-bucket",
    "config": {
        "provider": "cloudflare_r2",
        "account_id": "abc123",
        "bucket": "rubin-docs"
    },
    "credential_label": "r2"
}

// POST /orgs/rubin/services
{
    "label": "staging-bucket",
    "config": {
        "provider": "cloudflare_r2",
        "account_id": "abc123",
        "bucket": "rubin-docs-staging"
    },
    "credential_label": "r2"
}

// POST /orgs/rubin/services
{
    "label": "cdn",
    "config": {
        "provider": "cloudflare_workers",
        "account_id": "abc123",
        "zone_id": "xyz789"
    },
    "credential_label": "cloudflare"
}

// 3. Assign services to org slots
// PATCH /orgs/rubin
{
    "publishing_store_label": "docs-bucket",
    "staging_store_label": "staging-bucket",
    "cdn_service_label": "cdn"
}
```

#### Example: AWS + Fastly setup

```json
// Two credentials
// POST /orgs/spherex/credentials
{ "label": "aws", "credentials": { "provider": "aws", "access_key_id": "...", "secret_access_key": "..." } }

// POST /orgs/spherex/credentials
{ "label": "fastly", "credentials": { "provider": "fastly", "api_token": "..." } }

// Three services
// POST /orgs/spherex/services
{ "label": "s3-publish", "config": { "provider": "aws_s3", "bucket": "spherex-docs", "region": "us-east-1" }, "credential_label": "aws" }

// POST /orgs/spherex/services
{ "label": "fastly-cdn", "config": { "provider": "fastly", "service_id": "svc123" }, "credential_label": "fastly" }

// POST /orgs/spherex/services
{ "label": "dns", "config": { "provider": "route53", "hosted_zone_id": "Z1234" }, "credential_label": "aws" }

// PATCH /orgs/spherex
{ "publishing_store_label": "s3-publish", "cdn_service_label": "fastly-cdn", "dns_service_label": "dns" }
```

### Credential encryption

Docverse encrypts credential secrets at rest using [Fernet symmetric encryption](https://cryptography.io/en/latest/fernet/) from the `cryptography` library. Fernet provides AES-128-CBC encryption with HMAC-SHA256 authentication — ciphertext is tamper-evident and self-describing (the token embeds a timestamp and version byte). Encryption and decryption are in-process CPU-bound operations (sub-millisecond), requiring no external service calls or network round-trips. A single Fernet key is stored as a Kubernetes secret, never in the database, so database backups alone cannot decrypt credentials.

This approach avoids the operational complexity of Vault Transit (running a Vault instance, configuring Kubernetes auth, managing Vault policies, network round-trips for every encrypt/decrypt) for what amounts to encrypting a small number of short API tokens and keys. The `cryptography` library is already a transitive dependency via Safir.

Only the `encrypted_credentials` column in the `organization_credentials` table ({ref}`table-organization-credentials`) contains encrypted data.
Service configuration (bucket names, account IDs, zone IDs) in the `organization_services` table ({ref}`table-organization-services`) is stored as plaintext JSONB, since this information is non-secret and returned in API responses.

#### Key provisioning

The Fernet encryption key is provisioned through Phalanx's standard secrets management. In the application's `secrets.yaml`:

```{code-block} yaml
credential-encryption-key:
  description: >-
    Fernet key for encrypting organization credentials at rest.
  generate:
    type: fernet-key
```

Phalanx auto-generates the key, stores it in 1Password, and syncs it to a Kubernetes Secret. The key never appears in the database.

#### Key loading

At startup, the application loads the encryption key from environment variables sourced from the Kubernetes Secret:

- `DOCVERSE_CREDENTIAL_ENCRYPTION_KEY` — the current primary Fernet key.
- `DOCVERSE_CREDENTIAL_ENCRYPTION_KEY_RETIRED` (optional) — a retired key, present only during rotation periods.

When both keys are present, Docverse constructs a `MultiFernet([Fernet(primary), Fernet(retired)])`. `MultiFernet` tries decryption with each key in order, so credentials encrypted under either key are readable, while new encryptions always use the primary key. When only the primary key is present, Docverse still wraps it in `MultiFernet([Fernet(primary)])` to provide a uniform interface.

#### Python integration

`CredentialEncryptor` is a thin wrapper around `MultiFernet` that operates on raw bytes:

```{code-block} python
from cryptography.fernet import Fernet, MultiFernet


class CredentialEncryptor:
    """Encrypt and decrypt organization credentials using Fernet."""

    def __init__(
        self,
        *,
        current_key: str,
        retired_key: str | None = None,
    ) -> None:
        keys = [Fernet(current_key)]
        if retired_key is not None:
            keys.append(Fernet(retired_key))
        self._fernet = MultiFernet(keys)

    def encrypt(self, plaintext: bytes) -> bytes:
        """Encrypt a credential, returning a Fernet token."""
        return self._fernet.encrypt(plaintext)

    def decrypt(self, token: bytes) -> bytes:
        """Decrypt a Fernet token to recover the credential."""
        return self._fernet.decrypt(token)

    def rotate(self, token: bytes) -> bytes:
        """Re-encrypt a token under the current primary key."""
        return self._fernet.rotate(token)
```

All methods are synchronous — Fernet operations are sub-millisecond CPU-bound work, so no `async`/`await` is needed. The service layer JSON-encodes credential dicts to bytes before encryption and JSON-decodes after decryption. The plaintext is held only in memory for the duration of client construction.

In the factory pattern, `CredentialEncryptor` is a process-level singleton in `ProcessContext`. Since it holds no network connections or file handles, no shutdown cleanup is needed.

For testing, construct a `CredentialEncryptor` with `Fernet.generate_key()` — no mocking or external services required.

#### Key rotation

Key rotation uses `MultiFernet` to provide a zero-downtime transition:

1. **Generate a new Fernet key** in 1Password (or let Phalanx regenerate).
2. **Deploy with both keys**: set the new key as `DOCVERSE_CREDENTIAL_ENCRYPTION_KEY` and the old key as `DOCVERSE_CREDENTIAL_ENCRYPTION_KEY_RETIRED`. Restart pods. At this point, `MultiFernet` decrypts credentials under either key; new encryptions use the new key.
3. **Run the `credential_reencrypt` job** (scheduled periodically; see {ref}`periodic-job-scheduling`). The job iterates over all `organization_credentials` rows and calls `CredentialEncryptor.rotate()`, which re-encrypts each token under the current primary key. Unlike Vault's `vault:vN:` prefix, Fernet tokens don't indicate which key encrypted them, so the job processes all rows unconditionally. `MultiFernet.rotate()` is idempotent — re-encrypting an already-migrated token simply produces a new token under the same primary key.
4. **Remove the retired key**: once the re-encryption job completes, remove `DOCVERSE_CREDENTIAL_ENCRYPTION_KEY_RETIRED` and restart pods.

### Organization management

Organizations are created and configured via the Docverse API (not statically in Helm/Phalanx config). This keeps orgs in the same database as projects for consistency. The API has two tiers of admin endpoints:

- **Docverse superadmin APIs**: create/delete/list organizations (scoped via `superadmin_usernames` configuration)
- **Org admin APIs**: configure the org's settings, manage projects, manage edition rules

### Dashboard rendering

Docverse subsumes the role of LTD Dasher. Dashboard pages (`/v/index.html`) and custom 404 pages are rendered server-side using Jinja templates and project/edition metadata from the database, then uploaded as self-contained HTML files to the object store and served through the CDN. See the {ref}`dashboards` section for the full design.

Re-rendering is triggered by:

- Template repo changes (via GitHub webhook)
- Project metadata changes
- Edition updates (new build published, edition created/deleted)

Dashboard rendering is handled asynchronously via the task queue.

### Relationship to LTD Keeper v2

The LTD Keeper v2 Organization model (in `keeper/models.py`) established much of this design: the `OrganizationLayoutMode` enum (subdomain vs path), Fernet-encrypted credentials, org-scoped Tags and DashboardTemplates, and the Product→Organization foreign key. Key changes in Docverse:

- Cloud-agnostic storage (not AWS-only)
- Configurable CDN provider (not Fastly-only)
- GitHub-repo-based dashboard templates (not S3-bucket-stored)
- Gafaelfawr auth replacing the User/Permission model
- Infrastructure configuration at the org level, not the project level (see {ref}`projects`)
