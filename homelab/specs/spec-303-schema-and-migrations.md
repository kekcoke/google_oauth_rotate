---
id: SPEC-303
title: Postgres schema and migrations
status: draft
applies_to: [homelab]
depends_on: [SPEC-001, SPEC-302]
---

# SPEC-303 — Postgres schema and migrations

## Purpose

The multi-user token table, its indexes, and how it changes over time. The source material's
starting point is:

```sql
CREATE TABLE oauth_tokens (
  user_id UUID PRIMARY KEY,
  access_token TEXT,
  refresh_token TEXT,
  expires_at TIMESTAMP
);
```

This spec revises it in three ways that matter: the refresh token is **not** stored here (it is a
Vault path — [spec-302](spec-302-vault-integration.md)), `user_id` is Google's `sub` rather than a
generated UUID, and the columns the lifecycle actually needs are present.

## Definitions

- **`user_id`** — Google's `sub`. A string, not a UUID. Stable across email changes.
- **Migration** — a forward-only, versioned, idempotently-applied schema change.
- **Timestamptz** — `TIMESTAMP WITH TIME ZONE`. Plain `TIMESTAMP` is not acceptable here.

## Requirements

| ID | Level | Requirement |
|---|---|---|
| REQ-303-01 | MUST | Key `oauth_tokens` by `user_id` as `TEXT PRIMARY KEY`, holding Google's `sub`; MUST NOT use a generated UUID as the identity. |
| REQ-303-02 | MUST NOT | Include a refresh-token column; store `refresh_token_path` referencing Vault instead. |
| REQ-303-03 | MUST | Store `expires_at` as `TIMESTAMPTZ`; plain `TIMESTAMP` MUST NOT be used for any instant. |
| REQ-303-04 | MUST | Store `granted_scopes` as a text array (or equivalent) preserving the exact scope strings. |
| REQ-303-05 | MUST | Store `state`, constrained to the lifecycle's states, defaulting to the valid state. |
| REQ-303-06 | MUST | Store `refresh_token_issued_at`, `last_refresh_at`, `last_refresh_outcome`, `created_at`, `updated_at`. |
| REQ-303-07 | MUST | Store the Gmail watch fields — `watch_expiration`, `history_id` — nullable, since not every user has a watch. |
| REQ-303-08 | MUST | Encrypt the access-token column at rest, or store it in Vault; a plaintext access token MUST NOT be stored. |
| REQ-303-09 | MUST | Index `expires_at` so the sweep's due-token query does not scan the table. |
| REQ-303-10 | MUST | Index or constrain `state` so `DEAD` users can be excluded cheaply. |
| REQ-303-11 | MUST | Apply migrations forward-only, versioned, recorded in a migrations table, and idempotent on re-run. |
| REQ-303-12 | MUST | Make every migration safe to run against a live worker: no long exclusive locks on `oauth_tokens`, no destructive change without a documented two-step. |
| REQ-303-13 | MUST | Fail worker start-up when the schema version is unknown or newer than the code expects. |
| REQ-303-14 | MUST | Update `updated_at` on every row modification. |
| REQ-303-15 | MUST | Enforce that a row cannot exist without a `refresh_token_path`, so an access-token-only record is impossible at the database level. |
| REQ-303-16 | SHOULD | Provide a rollback note per migration describing how to reverse it, even though migrations are forward-only. |
| REQ-303-17 | SHOULD | Keep a `users` view or table for display attributes (email, display name) separate from the credential row. |

## Interface sketch

```sql
-- migrations/0001_oauth_tokens.sql
CREATE TABLE oauth_tokens (
  user_id                 TEXT PRIMARY KEY,           -- Google `sub` (REQ-303-01)
  email                   TEXT,                       -- display only
  access_token_enc        TEXT,                       -- encrypted (REQ-303-08)
  access_token_key_id     TEXT,
  refresh_token_path      TEXT NOT NULL,              -- Vault path (REQ-303-02, REQ-303-15)
  expires_at              TIMESTAMPTZ,                -- (REQ-303-03)
  granted_scopes          TEXT[] NOT NULL DEFAULT '{}',
  state                   TEXT NOT NULL DEFAULT 'VALID'
                            CHECK (state IN ('VALID','DEAD')),
  refresh_token_issued_at TIMESTAMPTZ,
  last_refresh_at         TIMESTAMPTZ,
  last_refresh_outcome    TEXT,
  watch_expiration        TIMESTAMPTZ,
  history_id              TEXT,
  created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX oauth_tokens_expires_at_idx ON oauth_tokens (expires_at)
  WHERE state = 'VALID';                              -- (REQ-303-09, REQ-303-10)

CREATE INDEX oauth_tokens_watch_expiration_idx ON oauth_tokens (watch_expiration)
  WHERE watch_expiration IS NOT NULL;
```

```js
// src/migrate.js
/** Applies pending migrations in order; idempotent (REQ-303-11). */
async function migrate(pool) {}
/** @throws {SchemaVersionError} unknown or too-new schema (REQ-303-13) */
async function assertSchemaVersion(pool, expected) {}
```

Note `state` is deliberately a small, closed set here. The richer lifecycle states in
[`token-lifecycle.md`](../../docs/token-lifecycle.md) are *derived* — `NEAR_EXPIRY` is a function
of `expires_at` and the clock, not a stored value that can go stale.

## Acceptance criteria

- [ ] A `pg_dump` contains no refresh token and no plaintext access token.
- [ ] Inserting a row without `refresh_token_path` is rejected by the database.
- [ ] The sweep's due-token query uses the index (verified with `EXPLAIN`).
- [ ] Running migrations twice is a no-op the second time.
- [ ] Migrations applied while the worker runs cause no failed token operations.
- [ ] A worker built for an older schema refuses to start against a newer one.
- [ ] `updated_at` advances on every modification.
- [ ] An `expires_at` written from one timezone reads back as the same instant in another.

## Tests

| Test ID | Covers | What it proves |
|---|---|---|
| T-303-01 | REQ-303-01 | `user_id` accepts a Google `sub` string; two users with one email are distinct rows. |
| T-303-02 | REQ-303-02 | The table has no refresh-token column; `refresh_token_path` is present. |
| T-303-03 | REQ-303-03 | An instant written in one session timezone reads back identically in another. |
| T-303-04 | REQ-303-04 | Scope strings round-trip exactly, including order-insensitive comparison. |
| T-303-05 | REQ-303-05 | An out-of-range `state` value is rejected. |
| T-303-06 | REQ-303-06 | All lifecycle timestamp columns round-trip. |
| T-303-07 | REQ-303-07 | Watch fields are nullable and round-trip when present. |
| T-303-08 | REQ-303-08 | The access-token column contains no plaintext, and carries a key id. |
| T-303-09 | REQ-303-09 | `EXPLAIN` on the due-token query shows an index scan on a seeded table. |
| T-303-10 | REQ-303-10 | Excluding `DEAD` users does not scan the table. |
| T-303-11 | REQ-303-11 | Migrating twice applies each migration once; the migrations table records them. |
| T-303-12 | REQ-303-12 | Applying migrations against a table under concurrent load causes no failed token operation. |
| T-303-13 | REQ-303-13 | A newer or unknown schema version prevents start-up with `SchemaVersionError`. |
| T-303-14 | REQ-303-14 | Any update advances `updated_at`. |
| T-303-15 | REQ-303-15 | Inserting without `refresh_token_path` is rejected at the database level, not only in code. |
| T-303-16 | REQ-303-16 | Every migration file carries a rollback note (checked structurally). |
| T-303-17 | REQ-303-17 | Display attributes live outside the credential row and can change without touching it. |

## Out of scope

- Vault paths and policies — [spec-302](spec-302-vault-integration.md).
- The store's programmatic interface — [spec-001](../../specs/spec-001-token-store.md).
- Backups and restore — [`../docs/deploy-homelab.md`](../docs/deploy-homelab.md).
- Postgres tuning, connection pooling sizes, and HA.

## References

- [`docs/token-lifecycle.md`](../../docs/token-lifecycle.md)
- [`specs/spec-001-token-store.md`](../../specs/spec-001-token-store.md)
