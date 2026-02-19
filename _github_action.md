(github-action)=

## GitHub Actions action (docverse-upload)

The `docverse-upload` action is a native JavaScript GitHub Action published to the GitHub Marketplace from the [lsst-sqre/docverse-upload](https://github.com/lsst-sqre/docverse-upload) repository.
It provides the same upload workflow as the Python client (see {ref}`client-server-monorepo`) but is purpose-built for GitHub Actions runners.

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

### Why native JavaScript instead of wrapping the Python client

A composite action that installs Python and then calls `docverse upload` would work, but carries overhead:

- **Python setup cost**: `actions/setup-python` adds 15–30 seconds to every job. For documentation builds that already have Python, this is free — but for projects using other languages, or for workflows where the docs build runs in a separate job, the setup time is wasted.
- **Node 20 is guaranteed**: GitHub Actions runners always have Node 20 available. A JavaScript action runs immediately with zero setup.
- **Native toolkit integration**: the `@actions/core` toolkit provides first-class support for step outputs, job summaries, annotations, and failure reporting. Wrapping a CLI subprocess requires parsing its output to surface these features.
- **Independent versioning**: the action is versioned via Git tags (`v1`, `v1.2.0`) following GitHub Actions conventions. It can release on its own cadence without coupling to the Python client's PyPI release cycle.

### Usage

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

### Inputs

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

### Outputs

| Output          | Description                            |
| --------------- | -------------------------------------- |
| `build-id`      | The Docverse build ID                  |
| `build-url`     | API URL of the created build           |
| `published-url` | Public URL where the edition is served |
| `job-status`    | Terminal queue job status              |
| `editions-json` | JSON array of all updated editions with slugs and published URLs (e.g., `[{"slug": "__main", "published_url": "https://pipelines.lsst.io/"}]`) |

(pr-comments)=

### Pull request comments

When `github-token` is provided, the action posts or updates a comment on the associated pull request summarizing the build and linking to all updated editions.
This gives PR reviewers immediate, clickable access to staged documentation without navigating the Docverse API or dashboards.

#### PR discovery

How the action finds the PR number depends on the workflow trigger event:

- **`pull_request` / `pull_request_target` events**: the PR number is read directly from `github.event.pull_request.number`.
- **`push` events**: the action queries the GitHub API (`GET /repos/{owner}/{repo}/pulls?head={owner}:{branch}&state=open`) to find open PRs for the pushed branch. If multiple PRs match, the action comments on all of them. If none match, the comment step is skipped silently.
- **Other events** (`workflow_dispatch`, `schedule`): the comment step is skipped silently.

#### Comment format

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

#### Comment deduplication

A hidden HTML marker `<!-- docverse:pr-comment:{org}/{project} -->` at the top of the comment body identifies the comment, scoped by organization and project.
On each build the action:

1. Lists existing comments on the PR and searches for the marker.
2. If found: updates the existing comment via `PATCH /repos/{owner}/{repo}/issues/comments/{comment_id}`.
3. If not found: creates a new comment via `POST /repos/{owner}/{repo}/issues/{pr_number}/comments`.

Multi-project PRs (repositories that publish to multiple Docverse projects) get one comment per project, each independently updated.

#### Edge cases

- **Job failed**: the comment reports the failure status and build ID instead of an edition table.
- **No editions updated**: the comment notes that no editions were updated and includes the build ID.
- **Partial failure** (`completed_with_errors`): successful editions appear in the main table; failed and skipped editions are listed in a collapsible `<details>` block.
- **Token lacks permissions**: the GitHub API returns 403; the action logs a warning via `core.warning()` but does not fail the step (the upload itself succeeded).
- **No PR context**: the comment step is skipped silently; the build proceeds normally.

#### Permissions

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

### Implementation details

(openapi-driven-development)=

#### OpenAPI-driven TypeScript development

The action's TypeScript types are generated from the Docverse server's OpenAPI spec, creating a cross-repo type safety chain:

1. **Pydantic models** in the `docverse` monorepo's client package define the API contract.
2. **FastAPI** generates an **OpenAPI spec** from those models; monorepo CI publishes the spec as a versioned artifact.
3. The `docverse-upload` repository pins a copy of the spec as `openapi.json`.
4. [`openapi-typescript`](https://openapi-ts.dev/introduction) generates **TypeScript types** (`generated/api-types.ts`) from the pinned spec.
5. [`openapi-fetch`](https://openapi-ts.dev/openapi-fetch/) provides a **type-safe HTTP client** that uses those generated types.

When the API contract changes, a developer updates `openapi.json` in the action repository (either manually or via a Dependabot-style automation).
Because the spec is committed, the diff in the pull request makes every schema change explicitly visible — field renames, added enum values, or new required properties are all reviewable before the action code is updated to match.
This provides a deliberate review gate that catches unintended breaking changes before they ship.

#### Build and bundle

The action is built with TypeScript and bundled into a single `dist/index.js` file using [`ncc`](https://github.com/vercel/ncc).
The bundled output is committed to the repository (standard practice for JavaScript GitHub Actions) so that the action runs without a `node_modules` install step.
The action targets the Node 20 runtime.

#### Tarball creation and upload

The action uses the Node.js [`tar`](https://www.npmjs.com/package/tar) package to create `.tar.gz` archives and computes a SHA-256 hash during creation.
The tarball is uploaded to the presigned URL via the Fetch API.

#### GitHub Actions integration

The action uses the `@actions/core` toolkit for runner integration:

- **Step summary**: on success, a Markdown summary is written to `$GITHUB_STEP_SUMMARY` showing the build ID, queue job status, and published URL.
- **Warning annotations**: if the queue job completes with warnings (partial success), the action emits warning annotations visible in the workflow run UI.
- **Step failure**: if the queue job fails, the action calls `core.setFailed()` with the failure reason, marking the step as failed.
- **Outputs**: build ID, build URL, published URL, and job status are set as step outputs for downstream workflow steps to consume.
- **PR comments**: when `github-token` is provided, posts a summary comment on the associated pull request with links to all updated editions (see {ref}`pr-comments`).

### Development workflow

#### Action development

The `docverse-upload` action uses a standard Node.js development workflow:

- `npm install` — install dependencies.
- `npm run generate-types` — regenerate `generated/api-types.ts` from `openapi.json` (runs `openapi-typescript`).
- `npm test` — run unit tests with Vitest.
- `npm run build` — compile TypeScript and bundle with `ncc` into `dist/index.js`.

#### Release workflow

- **GitHub Action** (`docverse-upload` repo): versioned via Git tags following GitHub Actions conventions (`v1`, `v1.2.0`). The `v1` tag is a floating major-version tag updated on each minor/patch release.

Housing the action in its own repository means its `v1`/`v1.2.0` tags reflect the action's own input/output contract and release cadence, independent of the server or client Python packages.
