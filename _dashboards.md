(dashboards)=

## Dashboard templating system

### Overview

Docverse renders dashboard pages (`/v/index.html`) and custom 404 pages server-side using Jinja templates, then uploads the rendered HTML to the object store as self-contained files with all assets inlined. This subsumes the role of LTD Dasher and gives organizations full control over the look and feel of their documentation portals.

Three tiers of templates, resolved in priority order:

1. **Project-level override**: a project can specify its own template source, allowing it to host its template in its own documentation repo or a dedicated repo.
2. **Org-level template**: each organization can configure a GitHub repo containing templates and assets shared across all projects in the org.
3. **Docverse built-in default**: a minimal, functional template bundled with the Docverse application, used when no org or project template is configured.

### Template source configuration

A template source is defined by four fields:

| Field          | Type | Default               | Description                                    |
| -------------- | ---- | --------------------- | ---------------------------------------------- |
| `github_owner` | str  | —                     | GitHub organization or user                    |
| `github_repo`  | str  | —                     | Repository name                                |
| `path`         | str  | `""` (repo root)      | Path within the repo to the template directory |
| `git_ref`      | str  | repo's default branch | Branch or tag to track                         |

This structure is flexible enough to support several layouts:

- **Dedicated template repo**: `lsst-sqre/docverse-templates`, path `""`, ref `main` — an org maintains a standalone repo for their templates.
- **Monorepo with multiple templates**: `lsst-sqre/docverse-templates`, path `rubin/` — multiple orgs share a repo but use different subdirectories.
- **Template in a project repo**: `lsst/pipelines_lsst_io`, path `.docverse/template/` — a project hosts its own template alongside its documentation source.

The same four fields appear on both the org-level and project-level template configuration. When a project has its own template source, it completely replaces the org template (no inheritance/merging of individual files).

### Template directory structure

At the root of the template directory (as specified by `path`), a `template.toml` file declares the template's contents:

```toml
[dashboard]
template = "dashboard.html.jinja"

[dashboard.assets]
css = ["style.css"]
js = ["filter.js"]
images = ["logo.svg", "favicon.png"]

[error_404]
template = "404.html.jinja"

[error_404.assets]
css = ["style.css"]
images = ["logo.svg"]
```

A minimal template directory might look like:

```
template.toml
dashboard.html.jinja
404.html.jinja          ← optional
style.css
logo.svg
filter.js
```

The `[error_404]` section is optional. If omitted, Docverse uses a built-in default 404 page. The Cloudflare Worker (or equivalent edge function) serves the 404 page when no object matches the requested path within a project.

### Asset inlining

All assets referenced in `template.toml` are inlined into the rendered HTML at render time, producing a single self-contained HTML file per page type (dashboard and 404). This eliminates the need for relative asset paths, simplifies CDN cache management (one URL to purge per page), and avoids asset/HTML version skew during updates.

The inlining strategy by file type:

- **CSS** (`.css`): inlined into `<style>` tags. Multiple CSS files are concatenated.
- **JavaScript** (`.js`): inlined into `<script>` tags. Multiple JS files are concatenated in declared order.
- **SVG** (`.svg`): inlined as raw SVG markup (preserving DOM interactivity and CSS styling).
- **Raster images** (`.png`, `.jpg`, `.gif`, `.webp`): base64-encoded as data URIs.

The renderer makes inlined assets available to the Jinja template as context variables. CSS and JS are provided as concatenated strings; images are provided as a dict keyed by filename (with dots and hyphens converted to underscores for template-friendly access):

```jinja
<head>
  <style>{{ assets.css }}</style>
</head>
<body>
  <header>{{ assets.images.logo_svg }}</header>
  <!-- ... -->
  <script>{{ assets.js }}</script>
</body>
```

For a typical dashboard page (CSS + a small JS filter script + an SVG logo + a favicon), the inlined HTML should be 30–80KB — well within reason for a metadata page that users visit occasionally.

### Jinja template context

The template receives a maximalist context with all available project and edition metadata. Since rendering happens in a background job (not on the request path), there is no performance concern with assembling a rich context.

#### Context structure

```python
@dataclass
class DashboardContext:
    org: OrgContext
    project: ProjectContext
    editions: EditionsContext
    assets: AssetsContext
    docverse: DocverseContext
    rendered_at: datetime

@dataclass
class OrgContext:
    slug: str
    title: str
    base_domain: str
    url_scheme: str                # "subdomain" or "path_prefix"
    published_base_url: str        # e.g. "https://lsst.io"

@dataclass
class ProjectContext:
    slug: str
    title: str
    doc_repo: str                  # GitHub repo URL
    published_url: str             # root URL, e.g. "https://pipelines.lsst.io"
    surrogate_key: str
    date_created: datetime

@dataclass
class EditionsContext:
    all: list[EditionContext]
    main: EditionContext | None     # the __main edition
    releases: list[EditionContext]  # kind=release, sorted semver descending
    drafts: list[EditionContext]    # kind=draft, sorted date_updated descending
    major: list[EditionContext]     # kind=major, sorted version descending
    minor: list[EditionContext]     # kind=minor, sorted version descending
    alternates: list[EditionContext]  # kind=alternate, sorted by title ascending

@dataclass
class EditionContext:
    slug: str
    title: str
    kind: str
    tracking_mode: str
    tracking_params: dict           # e.g. {"major_version": 2}
    published_url: str
    build: BuildContext | None
    date_created: datetime
    date_updated: datetime
    lifecycle_exempt: bool
    alternate_name: str | None      # deployment/variant name, if scoped

@dataclass
class BuildContext:
    id: str                         # Crockford Base32
    git_ref: str
    annotations: dict               # client-provided metadata
    uploader: str
    object_count: int
    total_size_bytes: int
    date_created: datetime
    date_uploaded: datetime
    alternate_name: str | None      # deployment/variant name, if any

@dataclass
class AssetsContext:
    css: str                        # concatenated CSS
    js: str                         # concatenated JS
    images: dict[str, str]          # filename_with_underscores → inlined content

@dataclass
class DocverseContext:
    api_url: str
    version: str
```

#### Pre-grouped editions

The `EditionsContext` provides both the flat `all` list and pre-grouped convenience lists so template authors can iterate by category without writing filter logic. Each group is pre-sorted in the most natural order for display:

- `releases`: semver descending (newest release first)
- `drafts`: `date_updated` descending (most recently active first)
- `major`/`minor`: version descending
- `alternates`: `title` ascending

#### Custom Jinja filters

The rendering environment registers utility filters:

- `timesince`: relative time display (e.g., "3 hours ago", "2 days ago")
- `filesizeformat`: human-readable byte sizes (e.g., "1.2 MB")
- `isoformat`: ISO 8601 datetime formatting
- `semver_sort`: sort a list of editions by semantic version

#### Template example

```jinja
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>{{ project.title }} — Documentation Editions</title>
  <style>{{ assets.css }}</style>
</head>
<body>
  <header>
    {{ assets.images.logo_svg }}
    <h1><a href="{{ project.published_url }}">{{ project.title }}</a></h1>
  </header>

  {% if editions.main and editions.main.build %}
  <section class="current">
    <h2>Current</h2>
    <a href="{{ editions.main.published_url }}">
      Latest ({{ editions.main.build.git_ref }})
    </a>
    <span class="meta">
      Updated {{ editions.main.date_updated | timesince }}
      · {{ editions.main.build.total_size_bytes | filesizeformat }}
    </span>
  </section>
  {% endif %}

  {% if editions.releases %}
  <section class="releases">
    <h2>Releases</h2>
    {% for edition in editions.releases %}
    <div class="edition-row">
      <a href="{{ edition.published_url }}">{{ edition.slug }}</a>
      <span class="ref">{{ edition.build.git_ref }}</span>
      <span class="date">{{ edition.build.date_uploaded | isoformat }}</span>
    </div>
    {% endfor %}
  </section>
  {% endif %}

  {% if editions.alternates %}
  <section class="alternates">
    <h2>Deployments</h2>
    {% for edition in editions.alternates %}
    <div class="edition-row">
      <a href="{{ edition.published_url }}">{{ edition.title }}</a>
      <span class="date">{{ edition.date_updated | timesince }}</span>
    </div>
    {% endfor %}
  </section>
  {% endif %}

  {% if editions.drafts %}
  <section class="drafts">
    <details>
      <summary>{{ editions.drafts | length }} draft(s)</summary>
      {% for edition in editions.drafts %}
      <div class="edition-row">
        <a href="{{ edition.published_url }}">{{ edition.slug }}</a>
        <span class="date">{{ edition.date_updated | timesince }}</span>
      </div>
      {% endfor %}
    </details>
  </section>
  {% endif %}

  <footer>
    Generated by <a href="{{ docverse.api_url }}">Docverse {{ docverse.version }}</a>
    at {{ rendered_at | isoformat }}
  </footer>
  <script>{{ assets.js }}</script>
</body>
</html>
```

Deployment-scoped draft editions carry an `alternate_name` field, which templates can use for filtering:

```jinja
{# Drafts for a specific deployment #}
{% for edition in editions.drafts if edition.alternate_name == "usdf-dev" %}
<div class="edition-row">
  <a href="{{ edition.published_url }}">{{ edition.slug }}</a>
  <span class="badge">usdf-dev</span>
</div>
{% endfor %}
```

### DashboardTemplate table

| Column         | Type                    | Description                                       |
| -------------- | ----------------------- | ------------------------------------------------- |
| `id`           | int                     | PK                                                |
| `org_id`       | FK → Organization       | Owning org                                        |
| `project_id`   | FK → Project (nullable) | If set, this is a project-level override          |
| `github_owner` | str                     | GitHub org/user                                   |
| `github_repo`  | str                     | Repository name                                   |
| `path`         | str                     | Path within repo (default `""` for root)          |
| `git_ref`      | str                     | Branch/tag to track                               |
| `store_prefix` | str (nullable)          | Object store prefix for current synced files      |
| `sync_id`      | str (nullable)          | Current sync version identifier (timestamp-based) |
| `date_synced`  | datetime (nullable)     | Last successful sync                              |

Unique constraint on `(org_id, project_id)` — at most one template configuration per org (where `project_id` is null) and one per project. See {ref}`table-dashboard-template` in the database schema section for the column reference within the full schema.

### Template sync

When a GitHub push webhook fires on a tracked template repo/ref, Docverse enqueues a `dashboard_sync` job. The sync process:

1. **Match webhook to templates**: query the `DashboardTemplate` table for rows matching the `github_owner/github_repo` and where the push ref matches the tracked `git_ref`. Check whether any changed files fall within the tracked `path` prefix. If no rows match, the webhook is ignored.

2. **Fetch template files**: use the GitHub Contents API (via the Safir GitHub App client or a Gafaelfawr-delegated token) to download the `template.toml` and all referenced template/asset files from the matched directory at the pushed commit SHA.

3. **Write to object store**: upload the fetched files to `__templates/{org_slug}/{sync_id}/` in the org's object store bucket, where `sync_id` is a timestamp-based identifier (e.g., `20260208T120000`). This creates a versioned snapshot alongside any previous syncs.

4. **Update database**: update the `DashboardTemplate` row with the new `store_prefix`, `sync_id`, and `date_synced`.

5. **Re-render dashboards**: for an org-level template, re-render dashboards for all projects in the org (parallelized via `asyncio.gather()`). For a project-level template, re-render only that project's dashboard. Each render reads the template files from the object store snapshot, assembles the Jinja context from the database, inlines assets, renders, and uploads the output HTML.

6. **Clean up previous sync**: the previous sync directory in the object store is marked for purgatory cleanup after a retention period. This provides rollback capability — if a new template is broken, an operator can revert the `store_prefix` in the database to the previous `sync_id` and re-render.

Docverse receives webhooks through two supported channels:

- **Direct GitHub App webhooks**: Docverse acts as a GitHub App using the Safir GitHub App framework, receiving push events directly.
- **Kafka via Squarebot**: Squarebot receives GitHub webhooks and publishes them to Kafka topics. Docverse consumes the relevant topics via FastStream. This allows sharing a GitHub App installation across multiple internal services.

Both channels feed into the same `dashboard_sync` job logic.

### Rendered output storage

The dashboard rendering pipeline produces several static files stored at well-known paths in the project's object store area:

- **Dashboard**: `{project_slug}/__dashboard.html`
- **404 page**: `{project_slug}/__404.html`
- **Version switcher**: `{project_slug}/__switcher.json`
- **Edition metadata**: `{project_slug}/__editions/{edition_slug}.json` (one per edition)

The double-underscore prefix on these filenames prevents collisions with edition slugs (which also use `__` prefix only for the reserved `__main` slug -- but `__main` is an edition slug used in URL paths like `/v/__main/`, not a filename at the project root).

The Cloudflare Worker (or equivalent edge function) maps URL paths to these files:

- `project.lsst.io/v/` or `project.lsst.io/v/index.html` → serves `{project_slug}/__dashboard.html`
- `project.lsst.io/v/switcher.json` → serves `{project_slug}/__switcher.json`
- `project.lsst.io/v/{edition}/_docverse.json` → serves `{project_slug}/__editions/{edition_slug}.json`
- Any request that resolves no object → serves `{project_slug}/__404.html` with a 404 status code

### Version switcher and edition metadata JSON

In addition to the dashboard HTML, Docverse generates static JSON files that client-side JavaScript in the published documentation can consume. These files are generated by the same rendering pipeline as the dashboard and are re-rendered on the same triggers.

#### Version switcher JSON (`__switcher.json`)

The [pydata-sphinx-theme version switcher](https://pydata-sphinx-theme.readthedocs.io/en/stable/user-guide/version-dropdown.html) loads a JSON file to populate its version dropdown. Docverse generates this file at `{project_slug}/__switcher.json`, served at `project.lsst.io/v/switcher.json`.

The file follows the pydata-sphinx-theme's expected format -- a JSON array of version entries:

```json
[
  {
    "name": "Latest (main)",
    "version": "__main",
    "url": "https://pipelines.lsst.io/",
    "preferred": true
  },
  {
    "name": "v2.3",
    "version": "v2.3",
    "url": "https://pipelines.lsst.io/v/v2.3/"
  },
  {
    "name": "v2.2",
    "version": "v2.2",
    "url": "https://pipelines.lsst.io/v/v2.2/"
  }
]
```

Field mapping from Docverse's edition model:

| Switcher field | Source                          | Description                                        |
| -------------- | ------------------------------- | -------------------------------------------------- |
| `name`         | edition title or formatted slug | Display label in the dropdown                      |
| `version`      | edition slug                    | Used by the theme for matching the current version |
| `url`          | edition `published_url`         | Link target                                        |
| `preferred`    | `true` for `__main` and `alternate` editions | Marks the recommended/stable version        |

By default, the switcher JSON includes the `__main` edition and all `release`, `major`, `minor`, and `alternate` editions, sorted with `__main` first, then alternates alphabetically, then by version descending. Draft editions are excluded by default to keep the dropdown focused on stable versions. `alternate` editions are included because they represent long-lived deployment targets that users need to navigate between. Both `__main` and `alternate` editions get `preferred: true`, since each represents a canonical view for its context.

This behavior is configurable via `template.toml`:

```toml
[switcher]
include_kinds = ["main", "release", "major", "alternate"]  # default; add "draft" to include drafts
```

In `conf.py`, projects point the pydata-sphinx-theme at the Docverse-generated file:

```python
    html_theme_options = {
        "switcher": {
            "json_url": "https://pipelines.lsst.io/v/switcher.json",
            "version_match": version,  # matches against the "version" field
        }
    }
```

#### Per-edition metadata JSON (`__editions/{slug}.json`)

Each edition gets a small metadata JSON file that client-side JavaScript in the published documentation can fetch to determine which edition is currently being viewed, whether it's the canonical version, and where the canonical version lives. These files are stored at `{project_slug}/__editions/{edition_slug}.json` and served at `project.lsst.io/v/{edition}/_docverse.json` via a Worker URL rewrite.

This approach stores the per-edition files in a separate object store path (`__editions/`) rather than injecting them into the build's immutable content. In pointer mode, the edition's URL space serves content from the build's object store prefix, so writing metadata into the build would violate immutability. The Worker recognizes requests for `_docverse.json` within an edition path and rewrites them to the corresponding `__editions/` file. In copy mode, the file can optionally be copied into the edition prefix alongside the build content.

Example for a draft edition:

```json
      "project": {
        "slug": "pipelines",
        "title": "LSST Science Pipelines",
        "published_url": "https://pipelines.lsst.io/"
      },
      "edition": {
        "slug": "DM-12345",
        "title": "DM-12345",
        "kind": "draft",
        "published_url": "https://pipelines.lsst.io/v/DM-12345/",
        "tracking_mode": "git_ref",
        "date_updated": "2026-02-08T12:00:00Z"
      },
      "canonical_url": "https://pipelines.lsst.io/",
      "is_canonical": false,
      "switcher_url": "https://pipelines.lsst.io/v/switcher.json",
      "dashboard_url": "https://pipelines.lsst.io/v/"
    }
```

Example for the `__main` (canonical) edition:

```json
{
  "project": {
    "slug": "pipelines",
    "title": "LSST Science Pipelines",
    "published_url": "https://pipelines.lsst.io/"
  },
  "edition": {
    "slug": "__main",
    "title": "Latest",
    "kind": "main",
    "published_url": "https://pipelines.lsst.io/",
    "tracking_mode": "git_ref",
    "date_updated": "2026-02-10T08:30:00Z"
  },
  "canonical_url": "https://pipelines.lsst.io/",
  "is_canonical": true,
  "switcher_url": "https://pipelines.lsst.io/v/switcher.json",
  "dashboard_url": "https://pipelines.lsst.io/v/"
}
```

The `canonical_url` always points to the `__main` edition's `published_url`. Client-side JavaScript can use this metadata to:

- Display "you're viewing a draft" or "you're viewing an older release" banners with a link to the canonical version.
- Inject `<link rel="canonical" href="...">` tags for SEO.
- Integrate with the version switcher to highlight the current edition.
- Show the edition kind and last-updated date in the page footer.

The `switcher_url` and `dashboard_url` fields provide stable references to the project's other Docverse-generated resources, so client JS doesn't need to construct URLs.

### Re-render triggers

Dashboard re-rendering is triggered by several events, all funneled through the task queue:

| Event                           | Trigger mechanism                     | Scope                              |
| ------------------------------- | ------------------------------------- | ---------------------------------- |
| Template repo push              | GitHub webhook → `dashboard_sync` job | All projects using that template   |
| Build processing completes      | Final step of `build_processing` job  | Single project                     |
| Edition created/deleted/updated | Final step of `edition_update` job    | Single project                     |
| Project metadata changed        | Enqueued by PATCH handler             | Single project                     |
| Manual re-render                | Admin API endpoint (if needed)        | Single project or all org projects |

For single-project re-renders triggered within other jobs (build processing, edition update), the render is performed inline as the final step — no separate job is enqueued. Only template syncs (which affect multiple projects) spawn their own `dashboard_sync` job.

Multiple triggers can race on the same project's dashboard files (e.g., two `build_processing` jobs, or a `build_processing` and a `dashboard_sync` job running concurrently). Docverse uses Postgres advisory locks at the project and edition level to serialize these writes. See {ref}`cross-job-serialization` for the locking strategy.
