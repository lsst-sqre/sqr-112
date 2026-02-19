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

Beyond the server-side architecture, Docverse's codebase includes a Python client library and CLI that share the repository with the server.
This monorepo structure and the client-owned model pattern are integral to maintaining type safety across the API boundary.

(client-server-monorepo)=

### Client-server monorepo

The Python client and the Docverse server share a single Git repository, following the monorepo pattern established by [Squarebot](https://github.com/lsst-sqre/squarebot).
The server package lives at the repository root (matching the [Ook](https://github.com/lsst-sqre/ook) and [Safir](https://github.com/lsst-sqre/safir) conventions), which simplifies Docker builds and makes `nox-uv`'s `@session` decorator work naturally:

```
docverse/                           # lsst-sqre/docverse
├── pyproject.toml                  # workspace root = server package
├── uv.lock                         # single lockfile for entire workspace
├── noxfile.py                      # shared dev tooling
├── ruff-shared.toml                # shared Ruff configuration
├── Dockerfile
├── client/
│   ├── pyproject.toml              # docverse-client (PyPI library)
│   └── src/
│       └── docverse/
│           └── client/
│               ├── __init__.py
│               ├── _client.py      # DocverseClient
│               ├── _cli.py         # CLI entry point
│               ├── _models.py      # Pydantic request/response models
│               └── _tar.py         # Tarball creation utility
├── src/
│   └── docverse/
│       ├── ...
│       └── models/                 # imports and extends client models
└── tests/
```

| Attribute     | Client                                | Server                                          |
| ------------- | ------------------------------------- | ----------------------------------------------- |
| PyPI name     | `docverse-client`                     | `docverse`                                      |
| Import path   | `docverse.client`                     | `docverse`                                      |
| Package style | Namespace package (`docverse.client`) | Namespace package (`docverse`)                  |
| Location      | `client/` subdirectory                | Repository root                                 |

The monorepo is motivated by four concerns:

- **Atomic model changes**: Pydantic models that define the API contract live in the client package. When an endpoint's request or response shape changes, the model update and the server handler update land in the same commit, eliminating version skew.
- **Shared dev tooling**: a single noxfile orchestrates linting, type-checking, and testing for both packages. The uv workspace handles editable installs of both packages automatically via `uv sync`.
- **Unidirectional dependency**: the server depends on the client package (for the shared models); the client never imports from the server. This keeps the client lightweight and installable in CI without pulling in FastAPI, SQLAlchemy, or Redis.
- **Lessons from LTD**: in the LTD era, the API server (`ltd-keeper`) and upload client (`ltd-conveyor`) lived in separate repositories with independently-defined models. The monorepo eliminates model drift issues between the two.

The `docverse-upload` GitHub Action is intentionally _not_ part of this monorepo.
The monorepo's "atomic model changes" benefit applies to Python packages that share Pydantic models at import time; the TypeScript action consumes the API through a generated OpenAPI spec, not through Python imports, so co-location provides no additional type-safety benefit.
Keeping the action in its own repository also avoids Git tag ambiguity — the monorepo already uses prefix-scoped tags (`client/v1.0.0`) for the Python client, and adding `v1`-style tags for the action would conflict with any future need for repository-level semver tags.
The OpenAPI spec serves as the contract bridge between the two repositories (see {ref}`openapi-driven-development`), and spec changes are explicitly visible in action-repo PR diffs, providing a clear review point for API evolution.

#### Shared noxfile

The monorepo uses a shared noxfile at the repository root.
Sessions that need the full locked environment use `nox_uv.session`, which calls `uv sync` to install both workspace members in editable mode along with all locked dependencies:

```{code-block} python
:caption: noxfile.py (excerpt — locked sessions)

from nox_uv import session

@session(uv_groups=["dev"])
def test(session: nox.Session) -> None:
    """Run server tests with locked dependencies."""
    session.run("pytest", "tests/")

@session(uv_groups=["typing", "dev"])
def typing(session: nox.Session) -> None:
    """Type-check both packages."""
    session.run("mypy", "src/", "client/src/", "tests/")
```

Because `uv sync` installs all workspace members as editable packages, changes to either package are reflected immediately — no manual `pip install -e` step is needed.

#### Dependency management

The monorepo uses a [uv workspace](https://docs.astral.sh/uv/concepts/workspaces/) with a single `uv.lock` at the repository root that replaces the traditional `server/requirements.txt` pattern.

**Workspace lockfile.**
A single `uv.lock` locks all dependencies for both the server and client packages.
uv computes the workspace's `requires-python` as the intersection of all members: if the client declares `>=3.12` and the server declares `>=3.13`, the lockfile resolves at `>=3.13`.
The client's `pyproject.toml` still declares `>=3.12` for PyPI consumers — the lockfile constraint only affects the development environment.

**Server** (`pyproject.toml` at the repository root):

- `requires-python = ">=3.13"` (single target version).
- Dependencies locked by `uv.lock` for reproducible Docker builds.
- [Dependency groups](https://docs.astral.sh/uv/concepts/dependency-groups/) for dev tooling: `dev` (test dependencies), `lint` (pre-commit, ruff), `typing` (mypy and stubs), `nox` (nox, nox-uv).
- `[tool.uv.sources]` maps `docverse-client` to the workspace member, so `uv sync` installs the local client in editable mode.

**Client** (`client/pyproject.toml`):

- `requires-python = ">=3.12"` (broad range for library consumers).
- Dependencies use broad version ranges appropriate for a PyPI library.
- [Dependency groups](https://docs.astral.sh/uv/concepts/dependency-groups/) mirror the server's pattern: `dev` (pytest, respx, and other test dependencies). Unlocked nox sessions read these groups via `nox.project.dependency_groups()` so the noxfile never duplicates dependency lists.

#### Testing matrix

The nox sessions use two distinct mechanisms to control dependency resolution:

| Session | Mechanism | Resolution | Python | Purpose |
| --- | --- | --- | --- | --- |
| `test` | `nox_uv.session` | Locked (`uv.lock`) | 3.13 | Server tests — same deps as Docker |
| `client-test` | `nox_uv.session` | Locked (`uv.lock`) | 3.13 | Client tests with lockfile deps |
| `client-test-compat` | `nox.session` + `session.install` | Highest (unlocked) | 3.12, 3.13 | Client as PyPI users install it |
| `client-test-oldest` | `nox.session` + `session.install` | `lowest-direct` | 3.12 | Validates client's lower bounds |
| `lint` | `nox_uv.session` | Locked (`uv.lock`) | 3.13 | Pre-commit hooks |
| `typing` | `nox_uv.session` | Locked (`uv.lock`) | 3.13 | mypy on both packages |

The key distinction: **locked sessions** use `nox_uv.session` (which calls `uv sync` under the hood); **compatibility sessions** use standard `nox.session` with `session.install()` (which calls `uv pip install`, bypassing the workspace lockfile entirely).
This separation lets the server pin exact versions for reproducibility while the client is tested against the same range of environments its PyPI users will encounter.

```{code-block} python
:caption: noxfile.py (excerpt — unlocked compatibility session)

import nox

CLIENT_PYPROJECT = nox.project.load_toml("client/pyproject.toml")

@nox.session(python=["3.12", "3.13"])
def client_test_compat(session: nox.Session) -> None:
    """Test the client with unlocked highest dependencies."""
    session.install(
        "./client",
        *nox.project.dependency_groups(CLIENT_PYPROJECT, "dev"),
    )
    session.run("pytest", "client/tests/")
```

#### Docker build

The Docker build uses a two-stage pattern that separates dependency installation (layer-cached) from application code:

```{code-block} dockerfile
:caption: Dockerfile (excerpt)

# Install locked dependencies (cached unless uv.lock changes)
COPY pyproject.toml uv.lock ./
COPY client/pyproject.toml client/pyproject.toml
RUN uv sync --frozen --no-default-groups --no-install-workspace

# Install workspace members without re-resolving
COPY client/ client/
COPY src/ src/
RUN uv pip install --no-deps ./client .
```

`uv sync --frozen --no-default-groups --no-install-workspace` installs only the locked production dependencies without development groups or the workspace packages themselves.
Both `pyproject.toml` files must be present for uv to validate the workspace structure.
The subsequent `uv pip install --no-deps` installs the actual workspace packages without triggering a new resolution.

#### Release workflow

- **Client** (`docverse` monorepo): released to PyPI on `client/v*` tags (e.g., `client/v1.2.0`). The GitHub Actions publish workflow runs `uv build --no-sources --package docverse-client` and uploads to PyPI. The `--no-sources` flag disables workspace source overrides so the built distribution references PyPI package names, not local paths.
- **Server** (`docverse` monorepo): released as a Docker image on bare semver tags (e.g., `1.2.0`), following the SQuaRE convention of omitting the `v` prefix. The GitHub Actions workflow builds and pushes the Docker image. Phalanx Helm charts reference the tagged image version.

#### Version metadata

Both packages use [setuptools_scm](https://setuptools-scm.readthedocs.io/) to derive their version from Git tags.
Because two independent tag namespaces coexist in one repository, each package scopes its tag matching with both `tag_regex` (for parsing) and a custom `describe_command` with `--match` (for discovery).
Both mechanisms must be used together — `tag_regex` alone is not sufficient because `git describe` would still pick up the wrong tag.

**Server** (`pyproject.toml` at repository root):

```{code-block} toml
:caption: pyproject.toml (server — version metadata)

[project]
dynamic = ["version"]

[tool.setuptools_scm]
fallback_version = "0.0.0"
tag_regex = '(?P<version>\d+(?:\.\d+)*)$'

[tool.setuptools_scm.scm.git]
describe_command = [
    "git", "describe", "--dirty", "--tags", "--long",
    "--abbrev=40", "--match", "[0-9]*",
]
```

- `--match "[0-9]*"` ensures `git describe` only considers bare semver tags, excluding `client/v*` tags.
- `tag_regex` matches bare `1.2.0` without a `v` prefix.
- `fallback_version` provides a version before the first tag exists.

**Client** (`client/pyproject.toml`):

```{code-block} toml
:caption: client/pyproject.toml (client — version metadata)

[project]
dynamic = ["version"]

[tool.setuptools_scm]
root = ".."
fallback_version = "0.0.0"
tag_regex = '^client/(?P<version>[vV]?\d+(?:\.\d+)*)$'

[tool.setuptools_scm.scm.git]
describe_command = [
    "git", "describe", "--dirty", "--tags", "--long",
    "--abbrev=40", "--match", "client/v*",
]
```

- `root = ".."` points setuptools_scm to the repository root, since `client/` is a subdirectory.
- `--match "client/v*"` ensures `git describe` only considers client-prefixed tags.
- `tag_regex` strips the `client/` prefix when parsing the version.
- The slash separator in the tag prefix (`client/v1.2.0`) avoids the dash-parsing ambiguity that setuptools_scm historically had with dashed prefixes (e.g., `client-v1.2.0` could be misinterpreted as a version component separator).

#### Changelog management

Both packages use [scriv](https://scriv.readthedocs.io/) (>= 1.8.0) with separate configuration files and fragment directories, so each package can collect changelog entries independently at its own release cadence.

**Layout:**

```
docverse/
├── scriv-client.ini          # scriv config for client
├── scriv-server.ini          # scriv config for server
├── client/
│   ├── changelog.d/          # client fragments
│   └── CHANGELOG.md          # client changelog
├── server-changelog.d/       # server fragments (root level)
└── CHANGELOG.md              # server changelog
```

The scriv configuration files use `.ini` format because scriv's `--config` flag only accepts `.ini` files, not TOML.
This means the configuration cannot live in the respective `pyproject.toml` files as `[tool.scriv]` sections.

```{code-block} ini
:caption: scriv-server.ini

[scriv]
fragment_directory = server-changelog.d
changelog = CHANGELOG.md
format = md
```

```{code-block} ini
:caption: scriv-client.ini

[scriv]
fragment_directory = client/changelog.d
changelog = client/CHANGELOG.md
format = md
```

Usage, wrapped in nox targets for convenience:

```{code-block} console
$ scriv create --config scriv-client.ini     # new client fragment
$ scriv create --config scriv-server.ini     # new server fragment
$ scriv collect --config scriv-client.ini --version 1.2.0   # collect at release
$ scriv collect --config scriv-server.ini --version 1.2.0   # collect at release
```

### Client-owned Pydantic models

The client package owns every Pydantic model that appears in API request and response payloads.
The server imports these models and can subclass them to add server-side concerns (database constructors, internal validators), but the wire format is always defined in the client.

```{code-block} python
:caption: docverse/client/_models.py

from pydantic import BaseModel, HttpUrl

class BuildCreate(BaseModel):
    """Request body for POST /orgs/:org/projects/:project/builds."""
    git_ref: str
    content_hash: str
    alternate_name: str | None = None
    annotations: dict[str, str] | None = None

class BuildResponse(BaseModel):
    """Response from build endpoints."""
    self_url: HttpUrl
    project_url: HttpUrl
    id: str
    status: str
    git_ref: str
    upload_url: HttpUrl | None = None
    queue_url: HttpUrl | None = None
    date_created: str
```

```{code-block} python
:caption: docverse/models/build.py

from docverse.client._models import BuildCreate as ClientBuildCreate
from ..db import Build as BuildRow

class BuildCreate(ClientBuildCreate):
    """Server-side build creation with DB integration."""

    async def to_db_row(self, project_id: int) -> BuildRow:
        return BuildRow(
            project_id=project_id,
            git_ref=self.git_ref,
            content_hash=self.content_hash,
            alternate_name=self.alternate_name,
        )
```

This pattern provides:

- **Single source of truth**: one model definition governs both serialization (client) and deserialization (server).
- **Automatic OpenAPI alignment**: FastAPI generates the OpenAPI spec from these Pydantic models. The TypeScript types for the GitHub Action are generated from that same spec (see {ref}`openapi-driven-development`), creating a chain of consistency from Python to TypeScript.
- **Safe refactoring**: renaming a field or adding a required attribute is a type error in both packages immediately, caught by mypy before CI even runs tests.

### Python client library

`DocverseClient` is an async HTTP client built on [httpx](https://www.python-httpx.org/) that implements the full upload workflow.
It follows HATEOAS navigation — the client discovers endpoint URLs from the API root and resource responses rather than hardcoding paths.

#### Key methods

| Method              | Description                                                       |
| ------------------- | ----------------------------------------------------------------- |
| `create_build()`    | `POST` to create a build record; returns the presigned upload URL |
| `upload_tarball()`  | `PUT` the tarball to the presigned URL                            |
| `complete_upload()` | `PATCH` to signal upload complete; returns the queue job URL      |
| `get_queue_job()`   | `GET` the queue job status                                        |
| `wait_for_job()`    | Poll the queue job with exponential backoff until terminal state  |

#### Upload workflow

```{code-block} python
:caption: Full upload using the Python client

from docverse.client import DocverseClient, create_tarball

async with DocverseClient(
    base_url="https://docverse.lsst.io",
    token=os.environ["DOCVERSE_TOKEN"],
) as client:
    # 1. Create a tarball from the built docs directory
    tarball_path, content_hash = create_tarball("_build/html")

    # 2. Create a build record (returns presigned upload URL)
    build = await client.create_build(
        org="rubin",
        project="pipelines",
        git_ref="tickets/DM-12345",
        content_hash=content_hash,
    )

    # 3. Upload the tarball to the presigned URL
    await client.upload_tarball(
        upload_url=build.upload_url,
        tarball_path=tarball_path,
    )

    # 4. Signal upload complete (enqueues processing)
    job_url = await client.complete_upload(build.self_url)

    # 5. Wait for processing to finish
    job = await client.wait_for_job(job_url)
    print(f"Build {build.id} processed: {job.status}")
```

#### Authentication

The client authenticates via a Gafaelfawr token passed either as the `token` constructor argument or through the `DOCVERSE_TOKEN` environment variable.
The token is sent as a bearer token in the `Authorization` header on every request.
For CI pipelines, this token is typically a bot token stored as a CI secret (see {ref}`auth` for how bot tokens are provisioned).

#### Tarball creation

The `create_tarball()` utility function creates a `.tar.gz` archive from a directory and computes its SHA-256 content hash:

```{code-block} python
def create_tarball(source_dir: str | Path) -> tuple[Path, str]:
    """Create a .tar.gz of source_dir; return (path, 'sha256:...' hash)."""
```

The tarball is written to a temporary file. The content hash is computed during archive creation (streaming through a `hashlib.sha256` digest) so the file is read only once.

#### Job polling

`wait_for_job()` polls the queue job endpoint with exponential backoff (initial interval 1 s, max interval 15 s, jitter).
It returns when the job reaches a terminal state (`completed` or `failed`).
On failure, it raises `BuildProcessingError` with the failure details from the job response.

### Command-line interface

The `docverse upload` CLI command wraps the Python client library for use in shell scripts and CI pipelines where the GitHub Action is not applicable (e.g., Jenkins, GitLab CI, local builds).

#### Usage

```{code-block} console
$ docverse upload \
    --org rubin \
    --project pipelines \
    --git-ref tickets/DM-12345 \
    --dir _build/html
```

#### Options

| Flag               | Env var              | Default                    | Description                               |
| ------------------ | -------------------- | -------------------------- | ----------------------------------------- |
| `--org`            | `DOCVERSE_ORG`       | —                          | Organization slug                         |
| `--project`        | `DOCVERSE_PROJECT`   | —                          | Project slug                              |
| `--git-ref`        | `DOCVERSE_GIT_REF`   | current Git HEAD           | Git ref for the build                     |
| `--dir`            | `DOCVERSE_DIR`       | —                          | Path to the built documentation directory |
| `--token`          | `DOCVERSE_TOKEN`     | —                          | Gafaelfawr authentication token           |
| `--base-url`       | `DOCVERSE_BASE_URL`  | `https://docverse.lsst.io` | Docverse API base URL                     |
| `--alternate-name` | `DOCVERSE_ALTERNATE` | —                          | Alternate name for scoped editions        |
| `--no-wait`        |                      | wait enabled               | Return immediately after signaling upload |

#### Exit codes

| Code | Meaning                                                  |
| ---- | -------------------------------------------------------- |
| 0    | Build uploaded and processed successfully                |
| 1    | Failure (authentication error, upload error, job failed) |
| 2    | Upload succeeded but job had warnings (partial success)  |

When `--no-wait` is used, exit code 0 means the upload was accepted and processing was enqueued — the CLI does not wait for the job to finish.
