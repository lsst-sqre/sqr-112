(auth)=

## Authentication and authorization

Docverse uses a two-layer authorization model:

1. **Ingress layer (Gafaelfawr)**: authenticates requests and enforces a baseline access scope for Docverse admin and Docverse API user access.
2. **Application layer (Docverse)**: performs fine-grained, org-scoped authorization using group memberships.

This approach keeps Gafaelfawr configuration minimal (no per-org ingress rules) while giving Docverse full control over org-level access policies that org admins can manage through the API without Phalanx Helm changes.

### Ingress layer

Two `GafaelfawrIngress` resources protect the Docverse API ({ref}`api` Ingress and authorization mapping). The general ingress requires the `exec:docverse` scope for all API routes and requests a **delegated internal token** so that Docverse can query Gafaelfawr's user-info API for the authenticated user's group memberships. A separate admin ingress requires the `admin:docverse` scope for superadmin routes.

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
---
apiVersion: gafaelfawr.lsst.io/v1alpha1
kind: GafaelfawrIngress
metadata:
  name: docverse-admin-ingress
config:
  scopes:
    all:
      - "admin:docverse"
  service: docverse
template:
  metadata:
    name: docverse-admin-ingress
  spec:
    rules:
      - http:
          paths:
            - path: /admin
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

The **Docverse superadmin** role is enforced at the ingress level. The `admin:docverse` scope is mapped from an identity-provider group via Gafaelfawr's `groupMapping` configuration. Routes that manage organizations (`/admin/*`) require this scope.

### Application-layer authorization

All org-scoped authorization is handled inside Docverse, using the authenticated user's **username** and **group memberships**.

#### Retrieving user metadata

On each request to an org-scoped endpoint, Docverse uses the delegated token from `X-Auth-Request-Token` to call Gafaelfawr's `/auth/api/v1/user-info` API. This returns the user's full metadata, including group memberships (sourced from LDAP, CILogon, or GitHub depending on the Gafaelfawr deployment). The response is cached within the request (in the factory) so a single request never calls the user-info API more than once.

#### Org membership table

Docverse maintains an `OrgMembership` table in its database that maps users and groups to roles within organizations:

| Column           | Type              | Description                      |
| ---------------- | ----------------- | -------------------------------- |
| `id`             | UUID              | Primary key                      |
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
| **superadmin** (global) | Create and manage organizations. Enforced via Gafaelfawr scope, not the OrgMembership table.                                                 |

No per-project ACLs — authorization is at the org level.

#### Role resolution

On each request to an org-scoped endpoint, Docverse resolves the user's effective role for the target organization:

1. Get the username from `X-Auth-Request-User`.
2. Get the user's group memberships from the Gafaelfawr user-info API (using the delegated token, cached within the request).
3. Query the `OrgMembership` table for all rows matching the username or any of the user's groups, scoped to the target org.
4. Take the highest-privilege role found (admin > uploader > reader).
5. If no matching rows exist, the user has no access to that org (403).

This resolution is implemented as a FastAPI dependency that handlers can require, receiving the resolved role (or raising 403).

#### Examples

- "Everyone in `g_spherex` is an uploader for the SPHEREx org": single `OrgMembership` row with `principal=g_spherex`, `principal_type=group`, `role=uploader`.
- "CI bot `docverse-ci-spherex` is an uploader for the SPHEREx org": single row with `principal=docverse-ci-spherex`, `principal_type=user`, `role=uploader`.
- "User `jdoe` is an admin for the Rubin org": single row with `principal=jdoe`, `principal_type=user`, `role=admin`.

Org admins manage membership through the Docverse API — no Phalanx Helm changes needed.

### CI bot tokens

For CI/CD pipelines (e.g., GitHub Actions uploading documentation), Gafaelfawr tokens are created using the [admin token creation API](https://gafaelfawr.lsst.io/api/rest.html#tag/admin/operation/post_admin_tokens_auth_api_v1_tokens_post) with a bot username (e.g., `bot-docverse-ci-rubin`). The resulting token string is stored as a GitHub Actions secret. The bot username is then added to the relevant org's `OrgMembership` table with the `uploader` role. See {ref}`clients` for how the Python client and GitHub Action consume these tokens.

### Testing

For testing, the `X-Auth-Request-User` header can be set directly (following the [Safir testing pattern](https://safir.lsst.io/user-guide/gafaelfawr.html)). The Gafaelfawr user-info API call can be mocked to return specific group memberships for test users. Safir provides a mock for this purpose via the `rubin-gafaelfawr` package.
