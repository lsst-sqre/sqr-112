(auth)=

## Authentication and authorization

Docverse uses a two-layer authorization model:

1. **Ingress layer (Gafaelfawr)**: authenticates requests and enforces a baseline access scope for Docverse admin and Docverse API user access.
2. **Application layer (Docverse)**: performs fine-grained, org-scoped authorization using group memberships.

This approach keeps Gafaelfawr configuration minimal (no per-org ingress rules) while giving Docverse full control over org-level access policies that org admins can manage through the API without Phalanx Helm changes.

### Ingress layer

A single `GafaelfawrIngress` resource protects the Docverse API ({ref}`api` Ingress and authorization mapping). The ingress requires the `exec:docverse` scope for all API routes (including `/admin/*`) and requests a **delegated internal token** so that Docverse can query Gafaelfawr's user-info API for the authenticated user's group memberships. Superadmin authorization is enforced at the application layer, not the ingress layer.

```yaml
apiVersion: gafaelfawr.lsst.io/v1alpha1
kind: GafaelfawrIngress
metadata:
  name: docverse-ingress
config:
  scopes:
    all:
      - "exec:docverse"
  service: docverse
  delegate:
    internal:
      scopes:
        - "exec:docverse"
template:
  metadata:
    name: docverse-ingress
  spec:
    rules:
      - http:
          paths:
            - path: /
              pathType: Prefix
              backend:
                service:
                  name: docverse
                  port:
                    name: http
```

Gafaelfawr provides the following headers to Docverse on each authenticated request:

- `X-Auth-Request-User`: the authenticated username
- `X-Auth-Request-Email`: the user's email (if available)
- `X-Auth-Request-Token`: the delegated internal token

The **Docverse superadmin** role is determined by the `DOCVERSE_SUPERADMIN_USERNAMES` environment variable, a comma-separated list of usernames. When a request arrives at an `/admin/*` route, Docverse checks the authenticated username (from `X-Auth-Request-User`) against this list before performing any `OrgMembership` query. This keeps superadmin management in application configuration rather than requiring Gafaelfawr scope or group mapping changes.

### Application-layer authorization

All org-scoped authorization is handled inside Docverse, using the authenticated user's **username** and **group memberships**.

#### Retrieving user metadata

On each request to an org-scoped endpoint, Docverse uses the delegated token from `X-Auth-Request-Token` to call Gafaelfawr's `/auth/api/v1/user-info` API. This returns the user's full metadata, including group memberships (sourced from LDAP, CILogon, or GitHub depending on the Gafaelfawr deployment). The response is cached within the request (in the factory) so a single request never calls the user-info API more than once.

#### Org membership table

Docverse maintains an `OrgMembership` table in its database that maps users and groups to roles within organizations:

| Column           | Type              | Description                      |
| ---------------- | ----------------- | -------------------------------- |
| `id`             | int               | Auto-increment primary key       |
| `org_id`         | FK → Organization | The organization                 |
| `principal`      | str               | A username or group name         |
| `principal_type` | enum              | `user` or `group`                |
| `role`           | enum              | `reader`, `uploader`, or `admin` |

See {ref}`table-org-membership` in the database schema section for the complete column reference.

A membership row can reference either a **username** (for individual grants, e.g., a CI bot user) or a **group name** (for group-based grants, matching the groups from Gafaelfawr/LDAP, e.g., `g_spherex`).

#### Roles

Three org-level roles, plus one global role:

| Role                    | Permissions                                                                                                                                  |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **reader**              | List and read projects, editions, builds within the org (read-only API access)                                                               |
| **uploader**            | Everything reader can do, plus create builds (upload documentation). Designed to be safe for CI tokens.                                      |
| **admin**               | Everything uploader can do, plus create/manage projects, configure org settings, manage editions and lifecycle rules, manage org membership. |
| **superadmin** (global) | Create and manage organizations. Determined by `superadmin_usernames` configuration. Super admins receive de facto `admin` role in every organization without explicit OrgMembership rows. |

No per-project ACLs — authorization is at the org level.

#### Role resolution

On each request to an org-scoped endpoint, Docverse resolves the user's effective role for the target organization:

1. Get the username from `X-Auth-Request-User`.
2. Check if the username is in the `superadmin_usernames` configuration. If so, return `admin` role immediately (basis: `super_admin`). No database query needed.
3. Get the user's group memberships from the Gafaelfawr user-info API (using the delegated token, cached within the request).
4. Query the `OrgMembership` table for all rows matching the username or any of the user's groups, scoped to the target org.
5. Take the highest-privilege role found (admin > uploader > reader).
6. If no matching rows exist, the user has no access to that org (403).

This resolution is implemented as a FastAPI dependency that handlers can require, receiving the resolved role (or raising 403).

#### Authorization basis tracking

The role resolution process tracks not just the granted role but *how* it was determined, providing operational observability through structured logging.

`AuthBasis` is an enum with three values:

| Value              | Meaning                                                     |
| ------------------ | ----------------------------------------------------------- |
| `super_admin`      | Username matched the `superadmin_usernames` configuration   |
| `user_membership`  | Role granted via a user-principal `OrgMembership` row       |
| `group_membership` | Role granted via a group-principal `OrgMembership` row      |

`AuthorizationResult` is a dataclass returned by the role resolution dependency:

| Field   | Type               | Description                                            |
| ------- | ------------------ | ------------------------------------------------------ |
| `role`  | `OrgRole`          | The effective role granted (`reader`, `uploader`, `admin`) |
| `basis` | `AuthBasis`        | How the role was determined                            |
| `group` | `str \| None`      | The group name when basis is `group_membership`        |

On each request, the following fields are bound to the structlog request logger:

- `username` — the authenticated user
- `auth_basis` — one of `super_admin`, `user_membership`, `group_membership`
- `auth_role` — the effective role
- `auth_group` — the group name (only when basis is `group_membership`)

This makes it straightforward to query logs for questions like "which requests were authorized via superadmin?" or "which group granted access to this user?"

#### Examples

- "Everyone in `g_spherex` is an uploader for the SPHEREx org": single `OrgMembership` row with `principal=g_spherex`, `principal_type=group`, `role=uploader`.
- "CI bot `docverse-ci-spherex` is an uploader for the SPHEREx org": single row with `principal=docverse-ci-spherex`, `principal_type=user`, `role=uploader`.
- "User `jdoe` is an admin for the Rubin org": single row with `principal=jdoe`, `principal_type=user`, `role=admin`.

Org admins manage membership through the Docverse API — no Phalanx Helm changes needed.

### CI bot tokens

For CI/CD pipelines (e.g., GitHub Actions uploading documentation), Gafaelfawr tokens are created using the [admin token creation API](https://gafaelfawr.lsst.io/api/rest.html#tag/admin/operation/post_admin_tokens_auth_api_v1_tokens_post) with a bot username (e.g., `bot-docverse-ci-rubin`). The resulting token string is stored as a GitHub Actions secret. The bot username is then added to the relevant org's `OrgMembership` table with the `uploader` role. See {ref}`client-server-monorepo` for how the Python client and {ref}`github-action` for how the GitHub Action consume these tokens.

### Testing

For testing, the `X-Auth-Request-User` header can be set directly (following the [Safir testing pattern](https://safir.lsst.io/user-guide/gafaelfawr.html)). The Gafaelfawr user-info API call can be mocked to return specific group memberships for test users. Safir provides a mock for this purpose via the `rubin-gafaelfawr` package.
