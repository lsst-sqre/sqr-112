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

### Organization configuration

Each organization owns:

- **Object store**: bucket name, credentials, provider (AWS S3, GCS, generic S3-compatible, or Cloudflare R2). The bucket is provisioned externally by the org admin; Docverse stores connection details.
- **Staging store** (optional): a separate object store bucket used for build tarball uploads, configured with the same shape as the publishing object store (provider, bucket, credentials). When configured, presigned upload URLs point to the staging bucket and the Docverse worker reads tarballs from it. When not configured, staging uses a `__staging/` prefix in the publishing bucket. The staging store optimization is useful when the publishing store is on a different network than the Docverse compute cluster -- for example, a GCS staging bucket in the same region as a GKE cluster paired with a Cloudflare R2 publishing bucket. The staging bucket only needs transient storage; tarballs are deleted after processing.
- **CDN**: provider choice (Fastly, Cloudflare Workers, Google Cloud CDN), service ID, API keys for cache purging. The CDN is provisioned externally; Docverse only interacts at runtime for cache invalidation and edge data store updates.
- **DNS**: For subdomain-based layouts, Docverse registers subdomains via DNS APIs (e.g., Route 53, Cloudflare DNS). When using Cloudflare, wildcard subdomains are supported on all plans via a proxied `*.domain` DNS record with free wildcard SSL.
- **URL scheme** (per-org setting, one of):
  - _Subdomain_: each project gets `project.base-domain` (e.g., `sqr-006.lsst.io`)
  - _Path-prefix_: all projects under a root path (e.g., `example.com/documentation/project`)
- **Base domain** (e.g., `lsst.io`) and **root path prefix** (for path-prefix mode).
- **Dashboard templates**: a GitHub repo containing Jinja templates and assets, configured at the org level with optional per-project overrides. See the [Dashboard Templating System section](#dashboards) for full details.
- **Edition slug rewrite rules**: an ordered list of rules that transform git refs into edition slugs. Configured at the org level with optional per-project overrides. See the {ref}`edition-slug-rewrite-rules` section for the full rule format.
- **Default edition lifecycle rules** ({ref}`projects`).

### Credential storage

Docverse encrypts organization credentials at rest using [Fernet symmetric encryption](https://cryptography.io/en/latest/fernet/) from the `cryptography` library. Fernet provides AES-128-CBC encryption with HMAC-SHA256 authentication — ciphertext is tamper-evident and self-describing (the token embeds a timestamp and version byte). Encryption and decryption are in-process CPU-bound operations (sub-millisecond), requiring no external service calls or network round-trips. A single Fernet key is stored as a Kubernetes secret, never in the database, so database backups alone cannot decrypt credentials.

This approach avoids the operational complexity of Vault Transit (running a Vault instance, configuring Kubernetes auth, managing Vault policies, network round-trips for every encrypt/decrypt) for what amounts to encrypting a small number of short API tokens and keys. The `cryptography` library is already a transitive dependency via Safir.

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

#### Database schema

Fernet tokens are self-describing (they embed a version byte, timestamp, IV, and HMAC), so the database schema needs no separate columns for nonces, key versions, or algorithm metadata:

```sql
CREATE TABLE organization_credentials (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    label TEXT NOT NULL,
    service_type TEXT NOT NULL,
    encrypted_credential TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, label)
);
```

The `label` is a human-friendly name (e.g., "Cloudflare R2 production"). The `service_type` identifies the provider (e.g., `cloudflare`, `aws_s3`, `fastly`). Credentials are write-only through the API — the GET response returns metadata (label, service type, timestamps) but never the decrypted value.

#### Python integration

`CredentialEncryptor` is a thin wrapper around `MultiFernet` that handles str↔bytes encoding:

```{code-block} python
from cryptography.fernet import Fernet, MultiFernet


class CredentialEncryptor:
    """Encrypt and decrypt organization credentials using Fernet."""

    def __init__(
        self,
        primary_key: str,
        retired_key: str | None = None,
    ) -> None:
        keys = [Fernet(primary_key)]
        if retired_key:
            keys.append(Fernet(retired_key))
        self._fernet = MultiFernet(keys)

    def encrypt(self, plaintext: str) -> str:
        """Encrypt a credential, returning a Fernet token."""
        return self._fernet.encrypt(
            plaintext.encode()
        ).decode()

    def decrypt(self, token: str) -> str:
        """Decrypt a Fernet token to recover the credential."""
        return self._fernet.decrypt(
            token.encode()
        ).decode()

    def rotate(self, token: str) -> str:
        """Re-encrypt a token under the current primary key.

        If the token is already encrypted under the primary key,
        the result is a fresh token (new IV and timestamp) under
        the same key. MultiFernet.rotate() is idempotent in the
        sense that calling it repeatedly always produces a valid
        token under the primary key.
        """
        return self._fernet.rotate(
            token.encode()
        ).decode()
```

All methods are synchronous — Fernet operations are sub-millisecond CPU-bound work, so no `async`/`await` is needed. The service layer calls `decrypt` when constructing an org-specific storage or CDN client; the plaintext is held only in memory for the duration of client construction.

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

- **Docverse superadmin APIs**: create/delete/list organizations (scoped via Gafaelfawr token scope)
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
