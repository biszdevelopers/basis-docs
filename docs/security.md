# Security Model

This page describes controls implemented in this repository. Deployment
controls such as TLS termination, network firewalls, backups, and monitoring
remain the operator's responsibility.

## OAuth and OpenID Connect

- Only authorization code flow is supported. `response_type=code` is required.
- S256 PKCE, `state`, and `nonce` are required for every authorization request.
- Redirect URIs must match registration exactly, except for the restricted
  `http://localhost:*/path` form available to public clients. Registrations
  must use HTTP or HTTPS and cannot embed credentials.
- Confidential clients authenticate with `client_secret_basic`; the secret is
  verified against a per-secret scrypt hash.
- Authorization codes are stored as hashes, expire after 60 seconds, and can
  be consumed once.
- Refresh tokens are stored as hashes. Rotation consumes the old token, and
  reuse revokes the token family.
- JWTs are signed with RS256. Resource APIs should validate issuer, audience,
  lifetime, algorithm, and `typ=at+jwt`.

## Request limits and headers

A process-local fixed-window limiter allows 120 requests per minute per IP and
route on `/oauth/token`, `/oauth/revoke`, `/oauth/authorize`,
`/oauth/interaction/:uid/consent`, `/oauth/callback/microsoft`,
`/oauth/clients/:clientId`, and `/api/me`. A rejected request gets `429` and
`Retry-After`. A multi-instance deployment does not share limiter state.

`X-Forwarded-For` and `X-Real-IP` are honored only when the socket peer is a
loopback or private proxy address. A direct public client cannot spoof the
limiter key, and a reverse proxy must overwrite forwarded headers with the
trusted client address.

The public and internal Hono apps use `secureHeaders()`. Picture responses also
set `Cross-Origin-Resource-Policy: same-site`. The stored picture endpoint is
public and is not a self-only route; use the private internal picture endpoint
when the caller must be authenticated with `INTERNAL_API_TOKEN`.

## Secrets and sessions

- Client secrets use scrypt with an individual random salt.
- Authorization codes, refresh tokens, and SSO session tokens are stored as
  SHA-256 hashes; raw values are returned or set only when first issued.
- The first configured private RSA JWK signs tokens. JWKS contains public
  versions of configured keys.
- `basis_sso` is an opaque random cookie backed by a database session row.
  Its idle and absolute limits default to 12 hours and 30 days.
- `OIDC_COOKIE_KEYS` values must be at least 32 characters; production requires
  two or more. Current HMAC uses the first key in the list.
- `INTERNAL_API_TOKEN` values that match the `.env.example` placeholder are
  rejected at startup.

## Upstream profile pictures

The Microsoft profile picture URL is user-influenced data. Fetches are limited
to HTTPS Microsoft hosts, at most three redirects with host validation on every
hop, a five-second timeout, and a 5 MB response cap. Unavailable or unsafe
pictures fall back to a generated identicon instead of being fetched.

## User and permission revocation

Users are identified by the stored Microsoft provider, issuer, and subject.
Disabling a user blocks future sign-ins, deletes their SSO sessions, revokes
refresh-token families, and sets `tokensValidAfter`. Re-enabling the user clears
that barrier, so any still-unexpired access token issued before the disable can
be accepted again.

The example API middleware verifies access tokens locally. To enforce immediate
per-user revocation, configure its `loadTokenSubject` callback to load
`disabled` and `tokensValidAfter` from a trusted source. Otherwise a locally
verified JWT remains usable until its `exp`.

Permissions are scoped to the client application. They are included in access
tokens as an array; changing a grant does not rewrite already issued JWTs.

## Internal management boundary

The internal listener requires `Authorization: Bearer <INTERNAL_API_TOKEN>`.
Management routes also require a user access token in
`X-Basis-Actor-Token`; the token must match the configured client, audience,
and scope. Route handlers check delegated permissions and application ownership.

Bind the listener to loopback or a protected private network. The shared token
can call the lower-level internal user and picture endpoints, so keep it out of
browsers and downstream apps.

There are no local password or MFA credential tables, sign-in or audit-log
tables, webhook delivery, portal lockout switch, or IP allowlist implemented in
this repository.
