# Getting Started

## Requirements

- Node.js 24 or newer and Bun 1.3.14.
- PostgreSQL 14 or newer.
- A Microsoft Entra application for upstream sign-in.

The repository includes an OIDC development demo, but it still needs Microsoft
credentials to complete a real sign-in.

## Prepare the service

Clone the repository, copy the example environment file, and install the
dependencies:

```bash
git clone https://github.com/basishacks/basis-auth.git
cd basis-auth
cp .env.example .env
bun install
```

Set these values in `.env` before starting:

| Variable | Value |
| --- | --- |
| `DATABASE_URL` | Connection string for your PostgreSQL database. |
| `OIDC_ISSUER` | Public origin of the provider, such as `http://localhost:3000`. Do not add a path, query, or fragment. |
| `OIDC_COOKIE_KEYS` | Comma-separated keys, each at least 32 characters. Replace the examples; placeholder values fail startup. Production requires at least two. |
| `INTERNAL_API_TOKEN` | Random secret of at least 32 characters for trusted internal API calls. |
| `MICROSOFT_ISSUER`, `MICROSOFT_CLIENT_ID`, `MICROSOFT_CLIENT_SECRET` | Set all three to enable Microsoft login. They are required in production and must be set together. |

Generate cookie keys with `openssl rand -hex 32`. In non-production,
`OIDC_JWKS_JSON` may be empty and an ephemeral RSA key is generated. Production
must set `OIDC_JWKS_JSON` or `OIDC_JWKS_FILE` to persistent private RSA JWKs.

The settings `MANAGEMENT_PORTAL_CLIENT_ID`, `MANAGEMENT_PORTAL_AUDIENCE`, and
`MANAGEMENT_PORTAL_SCOPE` are also required. They identify the user access token
accepted by the internal management API. Use the matching client, audience,
and scope for a connected portal; `.env.example` contains development values.

## Configure Microsoft Entra

Register this redirect URI in the Entra app:

```text
http://localhost:3000/oauth/callback/microsoft
```

Set `MICROSOFT_ISSUER`, `MICROSOFT_CLIENT_ID`, and
`MICROSOFT_CLIENT_SECRET` in `.env`. The service uses the configured Microsoft
origin and chooses a tenant authority from each client's `organizationIds`.
Microsoft login uses the provider's configured upstream scopes; downstream app
scopes do not become Microsoft Graph permissions.

## Prepare PostgreSQL and clients

Create the database, then apply the checked-in migrations:

```bash
createdb basis_auth
bun run db:migrate
```

The service also runs pending migrations on startup. For a new application,
register one resource audience and at least one client. Use the interactive
walkthrough:

```bash
bun run clients:add
```

A resource declares the audience and API scopes it accepts. A client declares
its redirect URIs, resources, allowed scopes, consent behavior, organizations,
and account filter. Confidential clients use a secret; leave it blank in the
walkthrough to generate an `sk-...` value. Copy the secret when printed because
the database stores only a scrypt hash.

The full menu is `bun run clients`. It supports listing, adding, editing,
disabling, removing clients, and registering resources. Database-managed
changes are available without a restart. Client metadata can remain cached for
up to 60 seconds, while a newly registered client is visible immediately. The
TUI prints the resolved database host, port, and name before writing.

For configuration-based registration, edit `OIDC_RESOURCES_JSON` and
`OIDC_CLIENTS_JSON` in `.env`. Startup upserts configured rows. Removing an
entry from JSON does not delete the database row.

If a client appears in the database but the running server returns `14001`,
check the build and database identity first:

```bash
curl -s https://auth.example/health
bun run clients:doctor <clientId> https://auth.example http://localhost:3000
```

`clients:doctor` compares the local database identity with each server, checks
whether the client row is visible to the running process, and warns when a
server is running an older build, when the databases differ, or when the client
was written after the server started. Clients disabled in the TUI return
`14002`; clients that do not exist return `14001`.

## Run locally

Set `NODE_ENV=development` and run:

```bash
bun run dev
```

This starts the IdP on port `PORT` (default `3000`), a Vite build watcher, and a
Vite development server (default `5173`). The Vite server proxies auth requests
to the IdP. Open the URL printed by Vite, then visit `/dev/demo` to run the
development-only OIDC flow.

The demo client and resource are created automatically in development. They
request `openid profile email`; Microsoft credentials and a domain mapped to an
organization are still required. Do not deploy the development demo publicly.

## Production run

Build and start the service:

```bash
bun run build
bun run start
```

For a public deployment, use an HTTPS issuer, persistent private JWKS, strong
cookie keys, Microsoft credentials, and a TLS-enabled PostgreSQL connection.
The runtime enforces persistent JWKS, Microsoft config, and at least two cookie
keys in production. Configure TLS at the reverse proxy or server boundary;
the issuer scheme itself is not enforced by config validation.

## Bootstrap grants

`BOOTSTRAP_PERMISSION_GRANTS_JSON` is an array of grants with `email`,
`applicationId`, and `permissions`, for example:

```json
[
  {
    "email": "admin@example.org",
    "applicationId": "admin-portal",
    "permissions": ["BiszPortal.Applications.read"]
  }
]
```

Matching grants are applied when that Microsoft identity is upserted during
sign-in. They are scoped to the named client application. They are not global
administrator roles and are not applied merely because the service starts.

## Next steps

- Read [Architecture](/architecture) for listeners, tokens, and persistence.
- Read [Wiring Up](/wiring-up) for exact client registration and porting steps.
- Read [Reference](/reference) for routes and environment variables.
