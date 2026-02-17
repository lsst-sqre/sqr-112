(organizations)=

## Organizations design

With _organizations_, a single Docverse deployment can host multiple documentation domains.
Doing so can enable a single institution (like AURA or NOIRLab) to host documentation for multiple multiple missions with completely separate white-labelled presentations.
It can also enable SQuaRE to provide Docverse-as-a-service for partner institutions.

The Organization is the sole infrastructure configuration boundary — all projects within an org share the same object store, CDN, root domain, URL scheme, and default dashboard templates.

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

Docverse uses HashiCorp Vault's [Transit secrets engine](https://developer.hashicorp.com/vault/docs/secrets/transit) for encryption-as-a-service. Vault never stores Docverse's secrets -- it encrypts and decrypts data sent to it, with encryption keys held inside Vault's barrier. This keeps Docverse cloud-agnostic (no dependency on GCP KMS or AWS KMS) and leverages the Vault instance already running in the RSP/Phalanx stack.

Docverse uses **direct encryption** (not envelope encryption) since the payloads are small secrets like API tokens and keys. The flow: send plaintext to Vault, receive a self-describing ciphertext token (e.g., `vault:v1:XjsPWPjq...`), store the ciphertext in Postgres. To decrypt, send the ciphertext token back to Vault. The `v1` version prefix tracks which key version was used, making rotation transparent.

Each organization gets its own named transit key (`docverse-org-{org_slug}`), providing tenant isolation and independent rotation schedules. Keys use the `aes256-gcm96` algorithm (Vault's default).

#### Vault authentication

Docverse authenticates to Vault using the [Kubernetes auth method](https://developer.hashicorp.com/vault/docs/auth/kubernetes). The pod's service account authenticates automatically, bound to a Vault policy scoped to Docverse's transit paths:

```{code-block} text
path "transit/encrypt/docverse-org-*" {
    capabilities = ["update"]
}
path "transit/decrypt/docverse-org-*" {
    capabilities = ["update"]
}
path "transit/keys/docverse-org-*" {
    capabilities = ["create", "read", "update"]
}
path "transit/rewrap/docverse-org-*" {
    capabilities = ["update"]
}
```

This grants Docverse encrypt/decrypt/rotate/rewrap on its own keys and nothing else.

#### Database schema

Vault's ciphertext token is self-describing (it embeds the key version, nonce, and algorithm), so the database schema is minimal -- no separate columns for nonces, DEKs, or key versions:

```sql
CREATE TABLE organization_credentials (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    label TEXT NOT NULL,
    service_type TEXT NOT NULL,
    vault_ciphertext TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, label)
);
```

The `label` is a human-friendly name (e.g., "Cloudflare R2 production"). The `service_type` identifies the provider (e.g., `cloudflare`, `aws_s3`, `fastly`). Credentials are write-only through the API -- the GET response returns metadata (label, service type, timestamps) but never the decrypted value.

#### Key rotation

Vault Transit supports key rotation without exposing plaintext at any point:

1. **Rotate the key**: creates a new key version. New encryptions use the latest version; old ciphertext remains decryptable.
2. **Rewrap existing ciphertext**: the `rewrap` endpoint re-encrypts ciphertext from the old key version to the new one without Vault ever returning the plaintext.
3. **Set minimum decryption version** (optional): once all ciphertext is rewrapped, retire old key versions.

Docverse implements rewrapping as a periodic background job (scheduled via Kubernetes CronJob; see {ref}`periodic-job-scheduling`) that iterates over all `organization_credentials` rows and rewraps any ciphertext not using the latest key version. Since the `vault:vN:...` prefix encodes the version, detecting stale ciphertext is a string prefix check.

#### Python integration

Docverse uses a thin async client built on `httpx` rather than the `hvac` library.
The rationale:

- Docverse is async throughout (FastAPI, async database access, async HTTP everywhere). `hvac` is synchronous, built on `requests`, and would require wrapping every call in `asyncio.to_thread`.
- `async-hvac` exists but has spotty maintenance and only provides generic `read`/`write` methods — no structured Transit API.
- The Vault Transit HTTP API is small and stable. A direct `httpx` wrapper covers all needed operations (create key, encrypt, decrypt, rotate, rewrap) in roughly 60 lines.
- `httpx` is already a Docverse dependency (used for webhook delivery and external API calls), so this avoids adding `hvac`/`requests` to the dependency tree.

```{code-block} python
import base64
from typing import Self

import httpx


class VaultTransitClient:
    """Async client for Vault Transit secrets engine."""

    def __init__(self, vault_addr: str, vault_token: str) -> None:
        self._client = httpx.AsyncClient(
            base_url=vault_addr.rstrip("/"),
            headers={"X-Vault-Token": vault_token},
            timeout=10.0,
        )

    async def aclose(self) -> None:
        await self._client.aclose()

    async def __aenter__(self) -> Self:
        return self

    async def __aexit__(self, *exc: object) -> None:
        await self.aclose()

    async def _post(self, path: str, **payload: object) -> dict:
        response = await self._client.post(
            f"/v1/transit/{path}", json=payload
        )
        response.raise_for_status()
        if response.status_code == 204:
            return {}
        return response.json()

    async def create_key(
        self, name: str, key_type: str = "aes256-gcm96"
    ) -> None:
        """Create a named encryption key (idempotent)."""
        await self._post(f"keys/{name}", type=key_type)

    async def encrypt(self, key_name: str, plaintext: str) -> str:
        """Encrypt plaintext, returning Vault ciphertext token."""
        b64 = base64.b64encode(plaintext.encode()).decode()
        result = await self._post(
            f"encrypt/{key_name}", plaintext=b64
        )
        return result["data"]["ciphertext"]

    async def decrypt(self, key_name: str, ciphertext: str) -> str:
        """Decrypt a Vault ciphertext token."""
        result = await self._post(
            f"decrypt/{key_name}", ciphertext=ciphertext
        )
        return base64.b64decode(
            result["data"]["plaintext"]
        ).decode()

    async def rotate_key(self, key_name: str) -> None:
        """Rotate to a new key version."""
        await self._post(f"keys/{key_name}/rotate")

    async def rewrap(
        self, key_name: str, ciphertext: str
    ) -> str:
        """Re-encrypt with latest key version without
        exposing plaintext."""
        result = await self._post(
            f"rewrap/{key_name}", ciphertext=ciphertext
        )
        return result["data"]["ciphertext"]
```

The same `httpx` client handles [Kubernetes auth](https://developer.hashicorp.com/vault/api-docs/auth/kubernetes#login) — a single `POST /v1/auth/kubernetes/login` with the pod's service-account JWT — plus periodic token renewal.
This keeps Vault interaction entirely within one lightweight async client rather than pulling in `hvac` solely for authentication.

In the factory pattern, `VaultTransitClient` is a process-level singleton in `ProcessContext` (one `httpx.AsyncClient` shared across requests), closed during application shutdown.
The service layer `await`s `decrypt` when it needs to construct an org-specific storage or CDN client; the plaintext is held only in memory for the duration of client construction.

The httpx-based client is straightforward to test: unit tests can use `httpx.MockTransport` or the [`respx`](https://lundberg.github.io/respx/) library to stub Vault responses, while integration tests can point at a dev-mode Vault instance (`vault server -dev`).

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
