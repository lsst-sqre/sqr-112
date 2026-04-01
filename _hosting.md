(documentation-hosting)=

## Documentation hosting

Like LTD before, Docverse decouple the documentation hosting from the API service.
The documentation host uses a simple and reliable cloud-based CDN and object store so that documentation is served with low latency and high availability around the world, even if the API service itself is down.
In LTD, we used Fastly as the CDN and AWS S3 for object storage.
With Docverse, we wish to support multiple hosting stacks, but also add a new hosting architecture using Cloudflare Workers + R2 that eliminates the S3 copy-on-publish bottleneck and reduces costs by an order of magnitude.

### The S3 copy-on-publish bottleneck

The original LSST the Docs platform serves documentation projects as wildcard subdomains (`pipelines.lsst.io`, `dmtn-139.lsst.io`) through Fastly's CDN.
LTD Keeper, a Flask-based REST API, manages three core entities: **products** (documentation projects), **builds** (individual CI-produced documentation snapshots), and **editions** (named pointers like `main` or `v1.0` that track specific builds).
Each entity has a corresponding path prefix in a shared S3 bucket.

Fastly VCL intercepts every request and performs regex-based URL rewriting to map the requested URL to an S3 object path.
For example, `pipelines.lsst.io/v/main/page.html` becomes `s3://bucket/pipelines/editions/main/page.html`.
This is elegant but rigid: it can only serve what's physically at the edition's S3 path.
So when the `main` edition is updated from build `b41` to `b42`, LTD Keeper must copy every object from `pipelines/builds/b42/` to `pipelines/editions/main/`.
The system then purges Fastly's cache using surrogate keys — which works well at ~150ms global propagation — but the S3 copy itself can take minutes for large documentation sets.

This copy-on-publish approach has the fundamental issue that edition updates are slow and can also be inconsistent for users during the copy window.
The original LTD implementation had the additional bug that an edition's objects would be deleted before the new build's objects were copied into place, causing 404s for pages that weren't in the Fastly cache.

The proposed solution replaces this with **edge-side dynamic resolution**: the CDN intercepts the request, extracts the project name and edition from the URL, consults an edge data store to determine which build the edition currently points to, and fetches the correct object directly from the build's storage path.
Edition updates become a metadata change (updating a key-value mapping) rather than a bulk data operation.

### The Cloudflare stack

Cloudflare can provide this edge-side dynamic edition resolution through a combination of its Workers edge compute platform, R2 object storage, and Workers KV key-value store.
Surprisingly, this platform is also substantially more cost-effective than the current Fastly + S3 architecture, even at large scale, due to R2's zero egress fees, free wildcard TLS certification and Workers' efficient edge execution model.

#### Architecture overview

With Cloudflare, a request is handled at the edge by a Worker script that parses the URL to determine the project and edition.
With Workers KV, the Worker looks up which build the edition currently points to, constructs the R2 key for the requested object, and fetches it directly from R2 using the native R2 bindings.
The Worker then returns the object with appropriate caching headers:

```{mermaid}
graph TD
    DNS["*.lsst.io<br/>(wildcard DNS → Cloudflare)"]
    DNS --> W1

    subgraph Worker["Cloudflare Worker"]
        W1["1. Parse Host header<br/>→ extract project"]
        W2["2. Parse URL path<br/>→ extract edition"]
        W3["3. KV lookup<br/>edition → build ID"]
        W4["4. Construct R2 key<br/>{project}/builds/{build}/{path}"]
        W5["5. Fetch from R2"]
        W6["6. Return with cache headers"]
        W1 --> W2 --> W3 --> W4 --> W5 --> W6
    end

    W3 -.-> KV[("Workers KV<br/>edition→build mappings")]
    W5 -.-> R2[("R2 Bucket<br/>Stores all builds once")]
    API["Docverse API"] -->|REST API writes| KV
```

With this architecture, builds are only stored in R2 in one place (`{project}/builds/{build_id}/`), and editions are just pointers to builds in Workers KV.
When an edition is updated, only the KV mapping changes — an instant metadata operation instead of a bulk S3 copy.

#### Worker request flow

The Worker implementation is approximately 100–200 lines of TypeScript.
The same Worker codebase supports both the **subdomain** and **path-prefix** URL schemes ({ref}`organizations`).
The Worker detects the scheme from the incoming hostname: if the hostname equals the organization's root domain (e.g., `docs.example.com`), it uses path-prefix routing; if the hostname is a subdomain of the root (e.g., `pipelines.lsst.io`), it uses subdomain routing.
Each organization gets its own Wrangler environment with route bindings that match only its URL scheme, so the Worker physically cannot receive requests for the wrong scheme (see {ref}`worker-environments` below).

**Subdomain routing** (e.g., `pipelines.lsst.io/v/main/page.html`):

1. **Parse Host header** to extract the project name from the subdomain (e.g., `pipelines.lsst.io` → `pipelines`).
2. **Parse URL path** to determine the edition and file path:
   - `/v/{edition}/page.html` → named edition
   - `/page.html` or `/` → default edition (`__main` by convention)
   - `/builds/{build_id}/page.html` → direct build access (bypasses KV lookup)

**Path-prefix routing** (e.g., `docs.example.com/pipelines/v/main/page.html`):

1. **Parse URL path** to extract the project name from the first path segment after the root prefix (e.g., `/pipelines/v/main/page.html` → `pipelines`).
2. **Parse remaining path** to determine the edition and file path, following the same patterns as subdomain routing.

After extracting the project, edition, and file path, both schemes converge on the same resolution logic:

3. **KV lookup**: read the edition→build mapping from Workers KV using key `{project}/{edition}`. KV reads are cached at the edge with a `cacheTtl` (e.g., 30 seconds) to reduce KV costs and latency on hot paths.
4. **Construct R2 key** from the `r2_prefix` field in the KV value (e.g., `pipelines/__builds/ABC123/`) concatenated with the file path. The R2 path convention is set by the publisher at write time, not baked into the Worker.
5. **Fetch from R2** via the native R2 binding (no HTTP overhead).
6. **Return response** with appropriate `Content-Type`, `Cache-Control`, and `Cache-Tag` headers. The Worker sets an `X-Docverse-Build` response header with the build ID for debugging.

**Directory index resolution**: when the R2 fetch returns a miss and the requested path has no file extension, the Worker tries appending `/index.html`. If that also misses, it returns a 404. For paths without a trailing slash that resolve to a directory index, the Worker issues a redirect to add the trailing slash. This eliminates the need for directory marker objects in R2 — the Worker's fallback chain handles index resolution at request time using R2's free `get()` operations.

The Worker also handles routing to the project dashboard page (and other Docverse metadata files) and returns appropriate 404 responses. See [dashboard templating system](#dashboards) for how dashboard pages are served.

(worker-environments)=

#### Cloudflare configuration

##### Software and deployment separation

The Worker codebase lives in the Docverse monorepo under `cloudflare-worker/`, co-located with the Python server and client packages.
This co-location ensures that changes to the KV key/value schema (written by the Python publisher, read by the Worker) land in the same commit, preventing contract drift.
The `cloudflare-worker/` directory is a self-contained TypeScript project with its own `package.json`, `tsconfig.json`, and a minimal `wrangler.toml`.

The `wrangler.toml` committed to the Docverse repo defines only the **software-level defaults** — the entry point, compatibility date, and binding names — without any org-specific configuration:

```toml
name = "docverse-router"
main = "src/index.ts"
compatibility_date = "2025-01-01"
```

Per-organization configuration (routes, KV namespace IDs, R2 bucket names, `URL_SCHEME`) lives in a separate **deployment repository** (`lsst-sqre/docverse-cloudflare-deployments` for Rubin Observatory).
This mirrors the separation between the Docverse server repo and Phalanx for Kubernetes deployments: the software repo owns the code, the deployment repo owns the configuration.

The deployments repo:

- Defines per-org `wrangler.toml` environment blocks with routes, KV/R2 bindings, and environment variables.
- Pins a specific ref of the Docverse repo to control when Worker code changes roll out.
- Provides a GitHub Actions workflow that checks out both repos, overlays the per-org config, and runs `wrangler deploy --env <org>`.
- Serves as a reference template for other institutions setting up Cloudflare Workers for their own Docverse organizations.

Deployments are triggered either by a push to the deployments repo (config change) or by a repository dispatch event from the Docverse repo (Worker code change).

##### Per-organization Wrangler environments

Each Docverse organization gets its own [Wrangler environment](https://developers.cloudflare.com/workers/wrangler/environments/) in the deployments repo, providing complete isolation of routes, KV namespaces, and R2 buckets.
This model ensures that a subdomain-based organization only receives subdomain traffic, a path-prefix organization only receives path-prefix traffic, and each organization's KV and R2 bindings are fully independent.
Deploying to a specific organization uses `wrangler deploy --env <org-env>`.

An example deployments-repo `wrangler.toml` for Rubin Observatory's organizations:

```toml
name = "docverse-router"
main = "src/index.ts"
compatibility_date = "2025-01-01"

# --- Subdomain-based organization (e.g., lsst.io) ---
[env.lsst-prod]
routes = [
  { pattern = "*.lsst.io/*", zone_name = "lsst.io" }
]

[[env.lsst-prod.kv_namespaces]]
binding = "EDITIONS"
id = "<LSST_PROD_KV_NAMESPACE_ID>"

[[env.lsst-prod.r2_buckets]]
binding = "DOCS_BUCKET"
bucket_name = "lsst-io-docs"

[env.lsst-prod.vars]
URL_SCHEME = "subdomain"

# --- Subdomain-based staging organization ---
[env.lsst-staging]
routes = [
  { pattern = "*.lsst-dev.io/*", zone_name = "lsst-dev.io" }
]

[[env.lsst-staging.kv_namespaces]]
binding = "EDITIONS"
id = "<LSST_STAGING_KV_NAMESPACE_ID>"

[[env.lsst-staging.r2_buckets]]
binding = "DOCS_BUCKET"
bucket_name = "lsst-dev-docs"

[env.lsst-staging.vars]
URL_SCHEME = "subdomain"

# --- Path-prefix organization (e.g., docs.example.com) ---
[env.example-prod]
routes = [
  { pattern = "docs.example.com/*", zone_name = "example.com" }
]

[[env.example-prod.kv_namespaces]]
binding = "EDITIONS"
id = "<EXAMPLE_PROD_KV_NAMESPACE_ID>"

[[env.example-prod.r2_buckets]]
binding = "DOCS_BUCKET"
bucket_name = "example-docs"

[env.example-prod.vars]
URL_SCHEME = "path-prefix"
ROOT_PATH_PREFIX = ""
```

The `URL_SCHEME` environment variable tells the Worker which routing logic to use, and the Wrangler route bindings ensure that only matching traffic reaches each environment.
This is belt-and-suspenders: even if the Worker received an unexpected request, the `URL_SCHEME` variable would cause it to parse the URL correctly for its configured scheme and return a 404 for anything that doesn't match.

##### DNS setup

For **subdomain-based** organizations: a proxied wildcard CNAME record (`*.lsst.io`) combined with an HTTP route pattern (`*.lsst.io/*`) directs all subdomain traffic through the Worker.
Cloudflare's Universal SSL automatically provisions and renews a wildcard certificate for `*.lsst.io` at no additional cost.
The underlying A record content is irrelevant (it is never reached) because the Worker intercepts all requests.

For **path-prefix** organizations: a single proxied A or CNAME record for the root domain (e.g., `docs.example.com`) with a route pattern (`docs.example.com/*`) directs traffic through the Worker.
Standard SSL applies (no wildcard needed).

##### Infrastructure provisioning

The KV namespace and R2 bucket for each organization are created via the Wrangler CLI:

```bash
# Create the KV namespace for an org environment
npx wrangler kv namespace create "EDITIONS" --env lsst-prod
npx wrangler kv namespace create "EDITIONS" --env lsst-prod --preview

# Create the R2 bucket
npx wrangler r2 bucket create lsst-io-docs
```

#### Two-tier caching strategy

The caching architecture uses two layers:

- **Workers KV** as the global source of truth for edition→build mappings, with an API for external writes that Docverse calls when an edition is updated. KV propagates updates globally within approximately **60 seconds** — acceptable for documentation that doesn't require sub-second freshness.
- **Per-PoP Cache API** as a hot local cache in front of KV reads (`cacheTtl` on KV read operations), eliminating KV costs and latency for frequently accessed mappings.

For the content itself, the Worker sets `Cache-Control` headers to layer browser and edge caching:

- **Browser cache**: short TTL (5 minutes, `max-age=300`) so users see updates relatively quickly.
- **Edge cache**: longer TTL (1 hour, `s-maxage=3600`) to reduce R2 read operations.

R2's built-in Tiered Read Cache automatically caches hot objects closer to users, providing an additional optimization layer.

#### Cache invalidation

When an edition is re-pointed to a new build, two things happen:

1. **KV is updated** — the new edition→build mapping takes effect within ~60 seconds globally.
2. **Edge cache is purged** — so users don't keep seeing the old build for the duration of the `s-maxage`.

Three cache purge strategies are available, depending on the Cloudflare plan:

- **Purge by hostname** (all plans): purge all cached content for a subdomain (e.g., `pipelines.lsst.io`). Slightly broader than necessary but fast and simple.
- **Purge by prefix** (all plans): purge by URL prefix (e.g., `pipelines.lsst.io/v/v23.0/`) for more targeted invalidation.
- **Purge by Cache-Tag** (Enterprise plan): surgical invalidation using `Cache-Tag` headers set by the Worker (e.g., `edition:pipelines/main`).

The complete "re-point edition" operation — a KV write plus a cache purge — replaces the S3 directory copy plus Fastly surrogate-key purge in the current LTD architecture.
Total time: under 2 seconds, versus minutes for the S3 copy approach.

#### Edition publishing pipeline

The KV write and cache purge are performed by a dedicated `publish_edition` background job, separate from the `build_processing` job that unpacks and uploads build artifacts.
This separation provides independent retry semantics: if the Cloudflare API is transiently unavailable, the build's artifacts are already safely in R2 and the edition's desired state is recorded in Postgres — only the edge publish needs retrying.

The publishing pipeline flows as follows:

1. **Build processing completes** — artifacts are uploaded to R2, edition tracking determines which editions should point to the new build, and the edition's `current_build_id` is updated in Postgres.
2. **`publish_edition` job is enqueued** — one job per affected edition. The job payload includes the edition ID, the target build ID, and the org's CDN service credentials.
3. **The job writes to KV** via the Cloudflare REST API, setting the `{project}/{edition}` key with the build metadata and `r2_prefix`.
4. **The job purges the edge cache** for the affected edition's URL space.
5. **On success**, the edition's `publish_status` is set to `synced` and the corresponding `edition_build_history` entry is annotated with the publish timestamp.

Any code path that changes an edition's build pointer — edition tracking, manual update, or rollback — enqueues a `publish_edition` job, ensuring all paths converge on the same publishing logic.

##### Publish status tracking

The edition model includes a `publish_status` field (enum: `synced`, `pending`, `failed`) that tracks whether the edge state matches the database state.
When a `publish_edition` job is enqueued, the status is set to `pending`.
On successful KV write, it transitions to `synced`.
If the job exhausts its retries, it transitions to `failed` and an alert is fired (e.g., Slack notification).

This status is exposed in the edition API response, so users and dashboards can see at a glance whether what's visible in the browser matches what the database intends.
The database always represents the **desired state** (which build an edition should point to), while KV represents the **actual state** (which build is being served).
The `publish_status` field makes any divergence between the two visible rather than hidden.

The `edition_build_history` table is also annotated with publish outcomes.
Each history entry records whether the corresponding build was successfully published to KV for that edition.
This provides a full audit trail for manual reconciliation if automated recovery fails: an operator can see that "edition X was intended to point to build 42 at time T, and that publish succeeded/failed."

##### Reconciliation loop

As a safety net, a periodic reconciliation job scans all editions, compares the database's `current_build_id` against the actual KV value, and re-enqueues `publish_edition` jobs for any that diverge.
This catches not only retry exhaustion but also silent failures, manual KV tampering, or incomplete publishes caused by server restarts mid-job.

The reconciliation loop runs on a configurable interval (e.g., every 5 minutes) and is lightweight — it reads KV values via the Cloudflare REST API's bulk-read endpoint and compares against the database state.
Editions already in `synced` status with matching KV values are skipped.

#### Wildcard subdomains and SSL

Wildcard subdomains work on **all Cloudflare plans**, including the free tier.
A proxied wildcard DNS record (`*.lsst.io`) combined with an HTTP route pattern (`*.lsst.io/*`) directs all subdomain traffic through a single Worker.
Cloudflare's Universal SSL automatically provisions and renews a wildcard certificate at no additional cost.

#### KV management

The KV namespace is the source of truth for which build each edition points to.
Docverse writes to it via the Cloudflare REST API whenever an edition is created, updated, or deleted.

**Key format**: `{project}/{edition}` (e.g., `pipelines/main`)

**Value format**: JSON with build metadata and the R2 prefix.
The `r2_prefix` field is set by the publisher at write time so that the Worker follows the pointer without needing to know the R2 path convention.
This decouples the storage layout from the Worker — if the R2 key structure ever changes, only the publisher needs updating.

```json
{
  "build_id": "ABC123",
  "r2_prefix": "pipelines/__builds/ABC123/",
  "git_ref": "main",
  "content_hash": "sha256:abc123...",
  "published_at": "2025-06-15T10:30:00Z"
}
```

**Single edition write** — the API call the `publish_edition` job makes:

```bash
curl -X PUT \
  "https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/storage/kv/namespaces/${KV_NAMESPACE_ID}/values/pipelines%2Fmain" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data-raw '{"build_id":"ABC123","r2_prefix":"pipelines/__builds/ABC123/","git_ref":"main","content_hash":"sha256:abc123...","published_at":"2025-06-15T10:30:00Z"}'
```

**Bulk write** — for migration or seeding all edition mappings at once (supports up to 10,000 key-value pairs per call):

```bash
curl -X PUT \
  "https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/storage/kv/namespaces/${KV_NAMESPACE_ID}/bulk" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data-raw '[
    {
      "key": "pipelines/main",
      "value": "{\"build_id\":\"ABC123\",\"r2_prefix\":\"pipelines/__builds/ABC123/\",\"git_ref\":\"main\",\"content_hash\":\"sha256:abc123...\",\"published_at\":\"2025-06-15T10:30:00Z\"}"
    },
    {
      "key": "pipelines/v23.0",
      "value": "{\"build_id\":\"DEF456\",\"r2_prefix\":\"pipelines/__builds/DEF456/\",\"git_ref\":\"v23.0\",\"content_hash\":\"sha256:def456...\",\"published_at\":\"2025-05-01T00:00:00Z\"}"
    }
  ]'
```

#### Cost model

R2's **zero egress fees** are the defining cost feature.
At 10 million requests per month, the estimated total cost is approximately **$6/month**.
Even at 100 million requests per month, the cost rises to roughly **$110/month** — still dramatically less than equivalent AWS infrastructure.

| Component               | 10M req/mo               | 100M req/mo |
| ----------------------- | ------------------------ | ----------- |
| Workers base + requests | $5.00                    | $32.00      |
| Workers KV reads        | $0.00 (within free tier) | $45.00      |
| Workers KV writes       | ~$0.05                   | ~$0.05      |
| R2 storage (50 GB)      | $0.60                    | $0.60       |
| R2 Class B ops (reads)  | $0.00 (within free tier) | $32.40      |
| Bandwidth (egress)      | **$0.00**                | **$0.00**   |
| **Total**               | **~$6**                  | **~$110**   |

### Other hosting options

Through configuration, Docverse can support multiple hosting stacks, allowing different organizations to choose their preferred CDN provider and architecture.
We can certainly continue to support the existing Fastly VCL and S3 architecture that we have used thusfar.
This sections surveys other hosting options evaluated, including Fastly Compute, CloudFront + Lambda@Edge and Google Cloud CDN.

#### Fastly Compute

Since LSST the Docs already runs on Fastly, migrating from VCL to Fastly Compute avoids changing CDN providers entirely.
Compute replaces VCL's domain-specific language with WebAssembly modules compiled from Rust, JavaScript, or Go, while retaining access to Fastly's identical cache infrastructure and **~150ms global surrogate-key purge** — the fastest in the industry.

Key characteristics:

- **Dynamic Backends** (GA since April 2023) allow the Wasm module to construct S3 URLs at runtime rather than pre-configuring every possible origin.
- The **Fastly KV Store** provides an edge-local key-value store for edition→build mappings, readable from any PoP with low latency.
- Wasm cold starts are **35 microseconds**, matching VCL's near-instant request processing.
- Existing surrogate key infrastructure carries over directly, preserving the current cache purging workflow.

One structural constraint: **VCL and Compute cannot coexist on the same Fastly service**.
Fastly recommends service chaining during transition — placing a Compute service in front of the existing VCL service, then gradually moving logic to Compute until the VCL service can be retired.

**Cost**: Fastly requires contacting sales for Compute pricing, with a **$50/month minimum** for paid accounts.
Bandwidth starts at **$0.12/GB** in North America (versus Cloudflare's $0.00/GB for R2 egress), making it roughly 5–30x more expensive than Cloudflare Workers + R2 at equivalent scale.

#### CloudFront + Lambda@Edge

CloudFront + Lambda@Edge supports dynamic routing through Lambda functions on the origin-request trigger (cache-miss only).
The CloudFront KeyValueStore (introduced in late 2024) offers a hybrid approach with sub-millisecond reads from a 5 MB key-value store accessible from CloudFront Functions.
However, Lambda@Edge functions must be deployed in us-east-1, logs are scattered across regional CloudWatch instances, cold starts range from 100–520ms, and cache invalidation takes ~2 minutes.
At approximately $443/month for 10 million requests, it is the most expensive option evaluated.

#### Google Cloud CDN

Google Cloud CDN provides no edge compute capability, restricting it to copy mode only.

(cdn-provider-evaluation)=

#### CDN provider comparison

| Criterion                  | Cloudflare Workers + R2 | Fastly Compute                   | CloudFront + Lambda@Edge                     |
| -------------------------- | ----------------------- | -------------------------------- | -------------------------------------------- |
| **Edge API calls**         | ✅ fetch(), 1000/req    | ✅ Dynamic backends              | ✅ Lambda@Edge only                          |
| **Wildcard subdomains**    | ✅ All plans, free SSL  | ✅ Paid, wildcard cert           | ✅ ACM wildcard                              |
| **Edge data store**        | KV (~60s propagation)   | KV Store (eventually consistent) | KeyValueStore (5 MB max)                     |
| **Cache purge speed**      | Seconds (URL purge)     | **~150ms** (surrogate key)       | ~2 minutes                                   |
| **Cold starts**            | None (V8 isolates)      | None (35μs Wasm)                 | 100–520ms (Lambda)                           |
| **Global PoPs**            | 330+                    | 80+                              | 600+ (but Lambda runs at ~13 regional edges) |
| **Egress cost**            | **$0.00**               | $0.12/GB                         | $0.085/GB                                    |
| **Monthly cost (10M req)** | **~$6**                 | ~$50–200                         | ~$443                                        |
| **Implementation effort**  | Low (~100–200 LOC TS)   | Medium (~200–400 LOC)            | High (multi-service orchestration)           |
| **Migration friction**     | CDN provider change     | Same provider, new runtime       | CDN provider change                          |
