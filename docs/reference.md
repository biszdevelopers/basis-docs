# Reference

## Public listener

The public listener defaults to `PORT=3000`.

| Method and route | Purpose |
| --- | --- |
| `GET /health` | Health response `{ "status": "ok" }`. |
| `GET /.well-known/openid-configuration` | OpenID Connect discovery. |
| `GET /.well-known/oauth-authorization-server` | OAuth authorization-server metadata. |
| `GET /oauth/authorize` | Authorization-code request and login/consent UI. S256 PKCE is mandatory. |
| `GET /oauth/interaction` | Pending interaction data for the built-in UI; requires the interaction cookie. |
| `POST /oauth/interaction/:uid/consent` | UI consent action; requires the interaction cookie and `x-csrf-token`. |
| `GET /oauth/upstream/microsoft` | Begins the upstream Microsoft redirect. |
| `GET /oauth/callback/microsoft` | Microsoft return URI. Register this exact path in Entra. |
| `GET /oauth/clients/:clientId` | Public client display information and registered owners. |
| `POST /oauth/token` | Authorization-code, refresh-token, and client-credentials grants. |
| `GET/POST /oauth/userinfo` | User claims for a valid user access token. |
| `GET /oauth/jwks` | Public RS256 signing keys. |
| `POST /oauth/revoke` | Revokes a refresh-token family when the token belongs to the authenticated client. |
| `GET/POST /oauth/logout` | Ends the basis-auth SSO session. It does not end the Microsoft session. |
| `GET /api/me` | Current basis-auth session identity; requires the SSO cookie. |
| `GET /api/picture/:userId` | Stored picture bytes; public to callers who know the user ID. |
| `GET /dev/demo` | Development-only sample OIDC client. |

Discovery and JWKS are cached for five minutes. The token endpoint accepts
form-encoded or JSON request bodies. Use HTTP Basic client authentication for
confidential authorization-code and refresh-token requests. Public clients
send `client_id` and no secret. The handler accepts a body `client_secret`
for the client-credentials grant; use HTTP Basic for confidential clients.

The discovery document advertises `client_secret_post`, but the implementation
does not accept a body secret for authorization-code or refresh-token grants.
Use `client_secret_basic` for those grants.

`/api/picture/:userId` sets a five-minute public cache and is not self-only.
The separate `/internal/users/:userId/picture` route requires the internal
bearer token.

## Internal listener

The internal listener defaults to `127.0.0.1:3001`. All routes require
`Authorization: Bearer <INTERNAL_API_TOKEN>`. Management routes also require
a delegated user token in `X-Basis-Actor-Token`; see [Management Integration
API](/portal).

- `GET /internal/users/:userId`, `GET /internal/users/:userId/picture`,
  `PATCH /internal/users/:userId`
- `GET /internal/users?page=…&pageSize=…`
- `GET/PATCH /internal/management/users/:userId`
- `/internal/applications` and its client, owner, resource, scope, permission,
  default-permission, credential, and redirect-URI routes

The management UI is not part of this repository.

## Environment variables

Variables with defaults are marked in the table. Session and database tuning
values are read directly from `process.env`.

| Variable | Purpose and validation |
| --- | --- |
| `NODE_ENV` | `development` (default), `test`, or `production`. Production requires Microsoft config, persistent JWKS, and two or more cookie keys. Development requires a loopback issuer. |
| `PORT` | Public HTTP port; default `3000`. |
| `DATABASE_URL` | PostgreSQL connection string; required. |
| `INTERNAL_API_HOST` | Internal listener bind address; default `127.0.0.1`. |
| `INTERNAL_API_PORT` | Internal listener port; default `3001`. |
| `INTERNAL_API_TOKEN` | Required, at least 32 characters. |
| `MANAGEMENT_PORTAL_CLIENT_ID` | Required client ID expected in delegated actor tokens. |
| `MANAGEMENT_PORTAL_AUDIENCE` | Required resource audience expected in delegated actor tokens. |
| `MANAGEMENT_PORTAL_SCOPE` | Required scope expected in delegated actor tokens; default `noesis.access`. |
| `OIDC_ISSUER` | Required URL origin. Paths, query strings, and fragments are rejected. |
| `OIDC_COOKIE_KEYS` | Comma-separated keys, each at least 32 characters; placeholder strings are rejected. Production requires at least two. |
| `OIDC_JWKS_JSON` / `OIDC_JWKS_FILE` | Private RSA JWKS. Generated ephemerally outside production; required in production. |
| `OIDC_CLIENTS_JSON` | JSON array of client registrations seeded on startup. |
| `OIDC_RESOURCES_JSON` | JSON array of resource-server audiences and supported scopes. |
| `BOOTSTRAP_PERMISSION_GRANTS_JSON` | JSON array of email, `applicationId`, and app-scoped permission grants. |
| `MICROSOFT_ISSUER`, `MICROSOFT_CLIENT_ID`, `MICROSOFT_CLIENT_SECRET` | Must be set together; required in production. |
| `ACCESS_TOKEN_LIFETIME_MS` | Positive whole seconds in milliseconds; default `600000` (10 minutes). |
| `CORS_ALLOWED_ORIGINS` | Optional comma-separated origins. Adds CORS headers to `/oauth/token` and `/oauth/userinfo` only for exact matches. |
| `SESSION_IDLE_TIMEOUT_MS` | SSO idle timeout; default `43200000` (12 hours). |
| `SESSION_ABSOLUTE_MAX_MS` | SSO absolute cap; default `2592000000` (30 days). |
| `DATABASE_POOL_MAX` | PostgreSQL pool size; default `10`. |
| `DATABASE_POOL_IDLE_TIMEOUT_MS` | Pool idle timeout; default `10000` ms. |
| `DATABASE_CONNECTION_TIMEOUT_MS` | Pool connection timeout; default `5000` ms. |
| `DATABASE_STATEMENT_TIMEOUT_MS` | Optional PostgreSQL statement timeout in milliseconds. |

`OIDC_CLIENTS_JSON` entries can define `clientId`, `name`,
`clientSecret`, `redirectUris`, `public`, `scopes`, `resourceScopes`,
`permissions`, `defaultPermissions`, `resources`, `requireConsent`,
`filterMode`, `filterContent`, and `organizationIds`.

Identity scopes are `openid`, `profile`, `email`, and `offline_access`.
Custom scopes belong to a resource server. Permission grants are app-specific
access-token claims, not a special OIDC scope.

## OAuth error codes

The numeric `code` field is an application code, not the HTTP status.

| Code | Error | Meaning |
| --- | --- | --- |
| `14001` | `invalid_client` / `invalid_request` | Client ID is unknown or unavailable. |
| `14003` | `invalid_client` | A public client sent a secret. |
| `14004` | `invalid_client` | Confidential client secret is missing or invalid. |
| `14005` | `invalid_client` | Client has no linked organizations. |
| `14100` | `invalid_request` | Redirect URI is missing or not registered. |
| `14401` | `invalid_scope` | Scope is not allowed by the client or resource. |
| `14407` | `unknown_resource` | Resource audience has no resource-server row. |
| `14429` | `unsupported_response_type` | Response type is not `code`. |
| `14501` | `invalid_target` | Resource is not registered for the client. |
| `2400` | `invalid_request` | Authorization interaction is missing or expired. |

Other OAuth errors, including `invalid_grant`, may have no numeric application
code. Read the `error` and `error_description` fields as well.
