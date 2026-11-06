# Management Integration API

This repository contains the provider-side internal API used by a trusted
management service. It does not contain the management portal UI or a public
`/api/users` administration API.

## Two-part authentication

Every internal request needs:

```http
Authorization: Bearer <INTERNAL_API_TOKEN>
```

The token is checked before routing and the response is marked
`Cache-Control: no-store`. The default listener is `127.0.0.1:3001`. Keep it
on loopback or a protected private network.

Routes under `/internal/management` and `/internal/applications` also require
a user access token in `X-Basis-Actor-Token`. The token must be a user token
from `MANAGEMENT_PORTAL_CLIENT_ID`, with the configured
`MANAGEMENT_PORTAL_AUDIENCE` and `MANAGEMENT_PORTAL_SCOPE`. Client-credentials
tokens are rejected. The actor's `permissions` claims and application ownership
are checked for each operation.

## User routes

| Route | Behavior |
| --- | --- |
| `GET /internal/users/:userId` | Returns the user row, including profile, `disabled`, and `tokensValidAfter`. |
| `GET /internal/users/:userId/picture` | Returns picture bytes, or 404 when absent. |
| `PATCH /internal/users/:userId` | Trusted-service patch for `displayName`, `email`, `disabled`, and `picture`. Disabling deletes SSO sessions and revokes refresh-token families. |
| `GET /internal/users?page=…&pageSize=…` | Paginated user list; requires `BiszPortal.Users.read` on the actor token. |
| `GET /internal/management/users/:userId` | User plus application affiliations, permission definitions, and grants; requires `BiszPortal.Users.read`. |
| `PATCH /internal/management/users/:userId` | Updates profile fields and per-application grants; requires `BiszPortal.Users.update`. |

The non-management user and picture routes require only the internal bearer
token. Treat that token as a high-trust credential. Management user routes also
reject attempts to disable the acting user's own account.

## Application routes

All application routes require both authentication headers. These delegated
permissions are checked in addition to application ownership for scoped routes.

| Route | Main action | Required permission |
| --- | --- | --- |
| `GET /internal/applications` | List caller-owned apps and organizations | `BiszPortal.Applications.read` |
| `POST /internal/applications` | Create an app registration | `BiszPortal.Applications.create` |
| `GET /internal/applications/:clientId` | View app details | `BiszPortal.Applications.read` |
| `PATCH /internal/applications/:clientId` | Change app name, consent, filters, organizations | `BiszPortal.Applications.update` |
| `DELETE /internal/applications/:clientId` | Delete the app and dependent OAuth data | `BiszPortal.Applications.delete` |
| `POST /internal/applications/:clientId/credentials` and `DELETE .../credentials` | Rotate or remove a secret | `BiszPortal.ApplicationCredentials.manage` |
| `POST /internal/applications/:clientId/owners`, `PATCH .../owners/:userId`, `DELETE .../owners/:userId` | Manage owners | `BiszPortal.ApplicationOwners.manage` |
| `POST /internal/applications/:clientId/resources/attach`; `POST/PATCH/DELETE .../resources` | Attach, create, edit, or remove resources | `BiszPortal.ApplicationResources.manage` |
| `PUT /internal/applications/:clientId/scopes` | Set client scopes per resource | `BiszPortal.ApplicationScopes.manage` |
| `POST/PATCH/DELETE /internal/applications/:clientId/permissions` | Edit the app permission catalog | `BiszPortal.ApplicationPermissions.manage` |
| `PUT /internal/applications/:clientId/default-permissions` | Set first-login defaults | `BiszPortal.ApplicationPermissions.manage` |
| `PUT /internal/applications/:clientId/redirect-uris` | Replace redirect URI list | `BiszPortal.ApplicationRedirectUris.manage` |

The service checks global delegated permissions and caller ownership for
client-specific changes. See `src/internal/app.ts` and
`src/internal/applications.ts` for request schemas and response shapes.

## Current feature boundary

The internal API supports users, client registrations, owners, resources,
resource scopes, permission catalogs and defaults, and redirect URIs. It does
not implement local password accounts, MFA reset, session-list/revoke routes,
consent-management routes, sign-in or audit logs, dashboard counters, webhook
alerts, or a portal lockout switch.
