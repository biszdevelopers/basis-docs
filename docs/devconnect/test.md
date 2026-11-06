# DevConnect Test Guide

This page describes the repository test commands and the optional PostgreSQL
integration suite. Docker is not required.

## Unit and route tests

The default suite needs no database:

```bash
bun install
bun run test
```

`bun run test` runs Vitest in single-run mode. The suite covers authorization,
token exchange, refresh rotation, revocation, UserInfo, identity assembly,
client caching, scope checks, signing keys, rate limiting, sessions, config,
Microsoft identity/error mapping, and web UI helpers.

Use watch mode during development:

```bash
bun run test:watch
```

## Coverage

Coverage uses `@vitest/coverage-v8` and is configured in `vitest.config.ts`.
Logging, rate limiting, scope coverage, client caching, and key handling have
100% line, branch, function, and statement thresholds.

## PostgreSQL integration tests

`src/oauth/service.integration.test.ts` runs only when both
`RUN_POSTGRES_TESTS=1` and `DATABASE_URL` are set. It connects to the supplied
PostgreSQL database and applies the checked-in migrations.

Use a dedicated disposable database. The suite inserts and updates test
registrations and users; do not point it at production or a shared database.

```bash
RUN_POSTGRES_TESTS=1 DATABASE_URL="postgresql://basis_auth:basis_auth@localhost:5432/basis_auth" bun run test
```

## Adding tests

Tests are colocated with the module they exercise and use `*.test.ts`
filenames. Name tests as behavior statements:

```ts
it("rejects code replay after a token has been issued", async () => {
  // ...
});
```

Add or update coverage when changing authorization, token, identity,
configuration, or migration behavior.
