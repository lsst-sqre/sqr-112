(code-architecture)=
## Code architecture

Docverse follows a hexagonal (ports-and-adapters) architecture where domain logic is isolated from infrastructure concerns.
Storage backends are swappable per-organization, and services interact only with protocol interfaces — never with concrete implementations directly.

### Layered architecture

Docverse follows the layered architecture pattern established in [Ook](https://github.com/lsst-sqre/ook):

- **dbschema**: SQLAlchemy models (database tables).
- **domain**: domain models (dataclasses/Pydantic), business logic, protocol definitions. Storage-agnostic.
- **handlers**: FastAPI route handlers, FastStream (Kafka) handlers. Thin — validate, resolve context, delegate to services.
- **services**: orchestration layer. Services coordinate storage backends and domain logic. Called by handlers.
- **storage**: interfaces to external data stores — databases, object stores, CDN APIs, DNS APIs, GitHub API.

CDN and object store abstractions live in the **storage** layer.
Protocol classes defining their interfaces live in the **domain** layer.

### Factory pattern

Following the [Ook factory pattern](https://github.com/lsst-sqre/ook/blob/main/src/ook/factory.py), Docverse uses a `Factory` class that is provided as a FastAPI dependency to handlers.
The factory combines process-level singletons (held in a `ProcessContext`) with a request-scoped database session to construct services and storage clients on demand.

For Docverse's multi-tenant architecture, the factory additionally handles **org-specific client construction**.
The flow:

1. A handler receives a request scoped to an organization (resolved from URL path or project slug).
2. The handler gets the `Factory` from FastAPI's dependency injection.
3. The factory loads the org's configuration from the database.
4. `factory.create_object_store(org)` inspects the org's provider config and returns the correct implementation (S3 client with that org's bucket/credentials, or a GCS client, etc.).
5. `factory.create_cdn_client(org)` does the same for the CDN provider.
6. `factory.create_edition_service(org)` wires up the org-specific object store, CDN client, and database stores into the edition update service.

Org-specific clients are cached within a request using a dict keyed by org ID inside the factory instance, so multiple operations on the same org within a single request reuse the same clients.

**Open design question — cross-request client pooling**: Org-specific clients (and their underlying connection pools) could potentially be cached in `ProcessContext` across requests, lazily created and keyed by org ID. This would avoid per-request connection setup overhead for frequently-accessed orgs. Needs exploration around credential rotation, memory, and stale-client concerns.

### Object store abstraction

A **protocol class** in the domain layer defines the interface that all object store implementations must satisfy.
**Concrete implementations** in the storage layer:

- _S3-compatible_: uses `aiobotocore` (async). Covers AWS S3, generic S3-compatible stores (MinIO, etc.), and Cloudflare R2 (which exposes an S3-compatible API).
- _GCS_: uses `gcloud-aio-storage` (async). GCS's API is distinct enough from S3 to warrant a separate implementation rather than forcing it through an S3-compatibility shim.

A **factory method** in the storage layer (called by the `Factory`) instantiates the correct implementation based on the org's provider configuration.
**Services** only interact with the protocol — they are backend-agnostic.

Cloudflare R2 is accessed via the S3-compatible implementation.
R2's S3 API compatibility is comprehensive enough that a dedicated implementation is not required.
The key difference is R2's zero egress fees, which is a cost consideration rather than an API difference.

#### Object store interface

The protocol defines these operations:

- **Individual operations**: upload object, copy object, move object (to purgatory prefix), delete object, get object metadata, generate presigned URL (for client uploads).
- **Bulk operations** (critical for performance): batch copy (for edition updates involving 500+ objects), batch move (for purgatory), batch delete (for hard-deleting purgatory contents). Bulk methods accept lists of objects and handle parallelism internally using asyncio semaphores. S3 implementations can additionally leverage S3-specific batch APIs where available.
- **Listing**: list objects under a key prefix.

### CDN abstraction

The CDN abstraction follows the same pattern: protocol in domain, implementations in storage, factory-constructed per org.

The CDN interface exposes **domain-meaningful operations** rather than provider-specific primitives:

- **Purge edition**: invalidate all cached content for a specific edition of a project.
- **Purge build**: invalidate cached content for a build (used during build deletion).
- **Purge dashboard**: invalidate the cached dashboard page for a project.
- **Update edition mapping**: write a new edition→build mapping to the edge data store (pointer mode only; no-op for copy-mode CDNs).

Each CDN implementation translates these operations into provider-specific API calls:

- **Fastly**: purge by surrogate key (a single API call invalidates all objects tagged with the edition's surrogate key). Edition mappings via Fastly KV Store API.
- **Cloudflare Workers**: purge by URL or embed build ID in cache keys so that mapping changes implicitly invalidate old entries. Edition mappings via Workers KV API.
- **Google Cloud CDN**: purge by URL pattern. No edge data store (copy mode only).

The edition/build/product domain objects (or their identifiers) are passed to the purge methods.
The implementation derives whatever provider-specific identifiers it needs (surrogate keys, URL prefixes, URL patterns, KV keys) from the domain objects and org configuration.

CDN implementations declare their capabilities via properties on the class.
The key capability is **pointer mode support** (`supports_pointer_mode`): CDNs with programmable edge compute and an edge data store can resolve the edition→build mapping at the edge, while CDNs without edge compute fall back to copy mode.
The edition update service checks `supports_pointer_mode` at runtime to select the pointer mode or copy mode code path.

### Queue backend abstraction

The queue backend follows the same protocol-based pattern as the object store and CDN abstractions.
A protocol class in the domain layer defines the interface for enqueuing jobs and querying job metadata, while concrete implementations in the storage layer adapt specific queue technologies.

The initial implementation uses **Arq via Safir's `ArqQueue`** with **Redis** as the message transport.
Safir provides both the production `ArqQueue` and a `MockArqQueue` for testing, which aligns with Docverse's existing testing patterns.
The `QueueJob` Postgres table remains the authoritative state store — the queue backend is treated as a delivery mechanism only.

See the {ref}`queue-backend-protocol` section in the queue design for the full protocol definition, implementation details, and infrastructure notes.

### Service layer

Services are the orchestration layer that ties storage backends, domain logic, and database stores together.
They are called by thin handlers that validate input, resolve context (e.g., which org the request targets), and delegate to the service.

For example, `EditionService` wires together the org-specific object store, CDN client, and database stores to coordinate edition updates.
When pointer mode is available, it writes a KV mapping and purges the CDN cache.
When copy mode is required, it computes a diff against the build inventory and performs the ordered in-place update.

Services are also invoked from background jobs processed by the {ref}`queue` system — the same service logic applies whether the caller is an HTTP handler or a queue worker.
