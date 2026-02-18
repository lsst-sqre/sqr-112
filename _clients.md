(clients)=

## Docverse clients

The Docverse server exposes a REST API ({ref}`api`) for managing organizations, projects, editions, and builds.
Two first-party clients consume that API: a **Python client library with CLI** for general-purpose use and CI pipelines, and a **native JavaScript GitHub Action** optimized for GitHub Actions workflows.
Both clients implement the full upload flow described in {ref}`projects` — create a build, upload a tarball, signal completion, and optionally poll the queue job to completion.
The Python client and server share a monorepo ([lsst-sqre/docverse](https://github.com/lsst-sqre/docverse)); the GitHub Action lives in its own repository ([lsst-sqre/docverse-upload](https://github.com/lsst-sqre/docverse-upload)).

### Repository layout

The Python client and the Docverse server share a single Git repository, following the monorepo pattern established by [Squarebot](https://github.com/lsst-sqre/squarebot):

```
docverse/                           # lsst-sqre/docverse
├── client/
│   ├── pyproject.toml              # docverse-client (PyPI)
│   └── src/
│       └── docverse/
│           └── client/
│               ├── __init__.py
│               ├── _client.py      # DocverseClient
│               ├── _cli.py         # CLI entry point
│               ├── _models.py      # Pydantic request/response models
│               └── _tar.py         # Tarball creation utility
├── server/
│   ├── pyproject.toml              # docverse (server package)
│   └── src/
│       └── docverse/
│           ├── ...
│           └── models/             # imports and extends client models
└── noxfile.py                      # shared dev tooling
```

| Attribute     | Client                                | Server                                |
| ------------- | ------------------------------------- | ------------------------------------- |
| PyPI name     | `docverse-client`                     | `docverse`                            |
| Import path   | `docverse.client`                     | `docverse`                            |
| Package style | Namespace package (`docverse.client`) | Namespace package (`docverse`)        |

The monorepo is motivated by four concerns:

- **Atomic model changes**: Pydantic models that define the API contract live in the client package. When an endpoint's request or response shape changes, the model update and the server handler update land in the same commit, eliminating version skew.
- **Shared dev tooling**: a single noxfile orchestrates linting, type-checking, and testing for both packages with `pip install -e client -e server`.
- **Unidirectional dependency**: the server depends on the client package (for the shared models); the client never imports from the server. This keeps the client lightweight and installable in CI without pulling in FastAPI, SQLAlchemy, or Redis.
- **Lessons from LTD**: in the LTD era, the API server (`ltd-keeper`) and upload client (`ltd-conveyor`) lived in separate repositories with independently-defined models. The monorepo eliminates model drift issues between the two.

The `docverse-upload` GitHub Action is intentionally _not_ part of this monorepo.
The monorepo's "atomic model changes" benefit applies to Python packages that share Pydantic models at import time; the TypeScript action consumes the API through a generated OpenAPI spec, not through Python imports, so co-location provides no additional type-safety benefit.
Keeping the action in its own repository also avoids Git tag ambiguity — the monorepo already uses prefix-scoped tags (`client/v1.0.0`) for the Python client, and adding `v1`-style tags for the action would conflict with any future need for repository-level semver tags.
The OpenAPI spec serves as the contract bridge between the two repositories (see {ref}`openapi-driven-development`), and spec changes are explicitly visible in action-repo PR diffs, providing a clear review point for API evolution.

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

### GitHub Actions action (docverse-upload)

The `docverse-upload` action is a native JavaScript GitHub Action published to the GitHub Marketplace from the [lsst-sqre/docverse-upload](https://github.com/lsst-sqre/docverse-upload) repository.
It provides the same upload workflow as the Python client but is purpose-built for GitHub Actions runners.

```
docverse-upload/                    # lsst-sqre/docverse-upload
├── action.yml
├── src/
│   └── index.ts
├── generated/
│   └── api-types.ts                # openapi-typescript output
├── openapi.json                    # pinned OpenAPI spec from docverse
├── package.json
├── tsconfig.json
└── dist/
    └── index.js                    # ncc-bundled output
```

#### Why native JavaScript instead of wrapping the Python client

A composite action that installs Python and then calls `docverse upload` would work, but carries overhead:

- **Python setup cost**: `actions/setup-python` adds 15–30 seconds to every job. For documentation builds that already have Python, this is free — but for projects using other languages, or for workflows where the docs build runs in a separate job, the setup time is wasted.
- **Node 20 is guaranteed**: GitHub Actions runners always have Node 20 available. A JavaScript action runs immediately with zero setup.
- **Native toolkit integration**: the `@actions/core` toolkit provides first-class support for step outputs, job summaries, annotations, and failure reporting. Wrapping a CLI subprocess requires parsing its output to surface these features.
- **Independent versioning**: the action is versioned via Git tags (`v1`, `v1.2.0`) following GitHub Actions conventions. It can release on its own cadence without coupling to the Python client's PyPI release cycle.

#### Usage

```{code-block} yaml
- name: Upload docs to Docverse
  uses: lsst-sqre/docverse-upload@v1
  with:
    org: rubin
    project: pipelines
    dir: _build/html
    token: ${{ secrets.DOCVERSE_TOKEN }}
```

Usage with PR comments enabled:

```{code-block} yaml
- name: Upload docs to Docverse
  uses: lsst-sqre/docverse-upload@v1
  with:
    org: rubin
    project: pipelines
    dir: _build/html
    token: ${{ secrets.DOCVERSE_TOKEN }}
    github-token: ${{ github.token }}
```

#### Inputs

| Input            | Required | Default                    | Description                                   |
| ---------------- | -------- | -------------------------- | --------------------------------------------- |
| `org`            | yes      | —                          | Organization slug                             |
| `project`        | yes      | —                          | Project slug                                  |
| `dir`            | yes      | —                          | Path to built documentation directory         |
| `token`          | yes      | —                          | Gafaelfawr token for authentication           |
| `base-url`       | no       | `https://docverse.lsst.io` | Docverse API base URL                         |
| `git-ref`        | no       | `${{ github.ref }}`        | Git ref (auto-detected from workflow context) |
| `alternate-name` | no       | —                          | Alternate name for scoped editions            |
| `wait`           | no       | `true`                     | Wait for processing to complete               |
| `github-token`   | no       | —                          | GitHub token for posting PR comments with links to updated editions. Typically `${{ github.token }}`. When omitted, PR commenting is disabled. |

#### Outputs

| Output          | Description                            |
| --------------- | -------------------------------------- |
| `build-id`      | The Docverse build ID                  |
| `build-url`     | API URL of the created build           |
| `published-url` | Public URL where the edition is served |
| `job-status`    | Terminal queue job status              |
| `editions-json` | JSON array of all updated editions with slugs and published URLs (e.g., `[{"slug": "__main", "published_url": "https://pipelines.lsst.io/"}]`) |

(pr-comments)=

#### Pull request comments

When `github-token` is provided, the action posts or updates a comment on the associated pull request summarizing the build and linking to all updated editions.
This gives PR reviewers immediate, clickable access to staged documentation without navigating the Docverse API or dashboards.

##### PR discovery

How the action finds the PR number depends on the workflow trigger event:

- **`pull_request` / `pull_request_target` events**: the PR number is read directly from `github.event.pull_request.number`.
- **`push` events**: the action queries the GitHub API (`GET /repos/{owner}/{repo}/pulls?head={owner}:{branch}&state=open`) to find open PRs for the pushed branch. If multiple PRs match, the action comments on all of them. If none match, the comment step is skipped silently.
- **Other events** (`workflow_dispatch`, `schedule`): the comment step is skipped silently.

##### Comment format

The comment uses a Markdown table to list all updated editions with their published URLs:

````markdown
<!-- docverse:pr-comment:rubin/pipelines -->
### Docverse documentation preview

| Edition | URL |
| ------- | --- |
| `__main` | https://pipelines.lsst.io/ |
| `DM-12345` | https://pipelines.lsst.io/v/DM-12345/ |

Build `01HQ-3KBR-T5GN-8W` processed successfully.
````

Edition data is extracted from the completed job's `editions_completed` progress array (see {ref}`queue`).
For partial failures (job status `completed_with_errors`), successful editions appear in the main table; failed and skipped editions are listed in a collapsible `<details>` block below.

##### Comment deduplication

A hidden HTML marker `<!-- docverse:pr-comment:{org}/{project} -->` at the top of the comment body identifies the comment, scoped by organization and project.
On each build the action:

1. Lists existing comments on the PR and searches for the marker.
2. If found: updates the existing comment via `PATCH /repos/{owner}/{repo}/issues/comments/{comment_id}`.
3. If not found: creates a new comment via `POST /repos/{owner}/{repo}/issues/{pr_number}/comments`.

Multi-project PRs (repositories that publish to multiple Docverse projects) get one comment per project, each independently updated.

##### Edge cases

- **Job failed**: the comment reports the failure status and build ID instead of an edition table.
- **No editions updated**: the comment notes that no editions were updated and includes the build ID.
- **Partial failure** (`completed_with_errors`): successful editions appear in the main table; failed and skipped editions are listed in a collapsible `<details>` block.
- **Token lacks permissions**: the GitHub API returns 403; the action logs a warning via `core.warning()` but does not fail the step (the upload itself succeeded).
- **No PR context**: the comment step is skipped silently; the build proceeds normally.

##### Permissions

The `github-token` requires `pull-requests: write` permission. Workflows must declare this explicitly:

```{code-block} yaml
permissions:
  pull-requests: write

steps:
  - name: Upload docs to Docverse
    uses: lsst-sqre/docverse-upload@v1
    with:
      org: rubin
      project: pipelines
      dir: _build/html
      token: ${{ secrets.DOCVERSE_TOKEN }}
      github-token: ${{ github.token }}
```

(openapi-driven-development)=

#### Implementation details

##### OpenAPI-driven TypeScript development

The action's TypeScript types are generated from the Docverse server's OpenAPI spec, creating a cross-repo type safety chain:

1. **Pydantic models** in the `docverse` monorepo's client package define the API contract.
2. **FastAPI** generates an **OpenAPI spec** from those models; monorepo CI publishes the spec as a versioned artifact.
3. The `docverse-upload` repository pins a copy of the spec as `openapi.json`.
4. [`openapi-typescript`](https://openapi-ts.dev/introduction) generates **TypeScript types** (`generated/api-types.ts`) from the pinned spec.
5. [`openapi-fetch`](https://openapi-ts.dev/openapi-fetch/) provides a **type-safe HTTP client** that uses those generated types.

When the API contract changes, a developer updates `openapi.json` in the action repository (either manually or via a Dependabot-style automation).
Because the spec is committed, the diff in the pull request makes every schema change explicitly visible — field renames, added enum values, or new required properties are all reviewable before the action code is updated to match.
This provides a deliberate review gate that catches unintended breaking changes before they ship.

##### Build and bundle

The action is built with TypeScript and bundled into a single `dist/index.js` file using [`ncc`](https://github.com/vercel/ncc).
The bundled output is committed to the repository (standard practice for JavaScript GitHub Actions) so that the action runs without a `node_modules` install step.
The action targets the Node 20 runtime.

##### Tarball creation and upload

The action uses the Node.js [`tar`](https://www.npmjs.com/package/tar) package to create `.tar.gz` archives and computes a SHA-256 hash during creation.
The tarball is uploaded to the presigned URL via the Fetch API.

##### GitHub Actions integration

The action uses the `@actions/core` toolkit for runner integration:

- **Step summary**: on success, a Markdown summary is written to `$GITHUB_STEP_SUMMARY` showing the build ID, queue job status, and published URL.
- **Warning annotations**: if the queue job completes with warnings (partial success), the action emits warning annotations visible in the workflow run UI.
- **Step failure**: if the queue job fails, the action calls `core.setFailed()` with the failure reason, marking the step as failed.
- **Outputs**: build ID, build URL, published URL, and job status are set as step outputs for downstream workflow steps to consume.
- **PR comments**: when `github-token` is provided, posts a summary comment on the associated pull request with links to all updated editions (see {ref}`pr-comments`).

### Development workflow

#### Shared noxfile

The monorepo uses a shared noxfile at the repository root.
Development sessions install both packages in editable mode:

```{code-block} python
:caption: noxfile.py (excerpt)

PIP_DEPENDENCIES = ["-e", "client", "-e", "server"]
```

This allows developers to run the full test suite (client unit tests, server integration tests) and have changes to either package reflected immediately.

#### Action development (docverse-upload repo)

The `docverse-upload` action uses a standard Node.js development workflow:

- `npm install` — install dependencies.
- `npm run generate-types` — regenerate `generated/api-types.ts` from `openapi.json` (runs `openapi-typescript`).
- `npm test` — run unit tests with Vitest.
- `npm run build` — compile TypeScript and bundle with `ncc` into `dist/index.js`.

#### Dependency management

The client and server follow different dependency strategies appropriate to their packaging:

- **Client** (library): dependencies are unpinned in `pyproject.toml` with broad compatibility ranges. As a library installed in user environments and CI runners, it must be compatible with a range of dependency versions.
- **Server** (application): dependencies are pinned via `uv pip compile` for reproducible Docker builds. The pinned requirements are regenerated in CI and committed as `server/requirements.txt`.

#### Release workflow

The client, server, and GitHub Action have independent release cycles:

- **Client** (`docverse` monorepo): released to PyPI on `client/v*` tags (e.g., `client/v1.2.0`). The GitHub Actions publish workflow runs `uv build` in the `client/` directory and uploads to PyPI.
- **Server** (`docverse` monorepo): deployed as a Docker image via Phalanx. Server releases are driven by Phalanx chart version bumps, not Git tags.
- **GitHub Action** (`docverse-upload` repo): versioned via Git tags following GitHub Actions conventions (`v1`, `v1.2.0`). The `v1` tag is a floating major-version tag updated on each minor/patch release.

Housing the action in its own repository means its `v1`/`v1.2.0` tags reflect the action's own input/output contract and release cadence, independent of the server or client Python packages.
