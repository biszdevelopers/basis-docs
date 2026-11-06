# Database Setup

The IdP connects to PostgreSQL through `DATABASE_URL`. The public and internal
listeners are part of the same service process and share this connection. The
management portal UI, when deployed, should use the private internal API rather
than connecting directly to PostgreSQL.

## Stored data

The current schema includes:

- Microsoft-backed user identity, profile fields, disabled state, and token
  revocation barrier.
- Organizations and accepted email-domain suffixes.
- OIDC clients, resources, per-client permission definitions, defaults, and
  user grants.
- SSO sessions, authorization requests, one-time authorization codes, refresh
  token families, and remembered consent.
- Upstream Microsoft authorization state, verifier, nonce, and expiry.

There are no local password or MFA tables, audit-event tables, or sign-in-log
tables in the current schema.

## Local development

Use a PostgreSQL 14+ database. Docker is not required.

```bash
createdb basis_auth
DATABASE_URL="postgresql://<user>:<password>@localhost:5432/basis_auth" bun run db:migrate
```

Set the same connection string in `.env`:

```text
DATABASE_URL=postgresql://<user>:<password>@localhost:5432/basis_auth
```

## Production setup

Create a database and a restricted application role. Apply migrations using
the same `DATABASE_URL` that the service will use:

```sql
CREATE ROLE basis_auth LOGIN PASSWORD 'long-random-password';
CREATE DATABASE basis_auth OWNER basis_auth;
```

```bash
DATABASE_URL="postgresql://basis_auth:password@db.example:5432/basis_auth?sslmode=require" bun run db:migrate
```

Restrict database network access, require TLS where supported, and schedule
backups. The process uses the database role behind `DATABASE_URL` for both
listeners.

## Migrations

Edit `src/database/schema.ts`, generate a new migration, review its SQL, then
apply it:

```bash
bun run db:generate
bun run db:migrate
```

Do not edit historical migrations. `src/index.ts` applies pending checked-in
migrations during service startup. Test production migration plans against a
copy of the database before rolling them out.
