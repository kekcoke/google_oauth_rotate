---
id: SPEC-001
title: Token store
status: draft
applies_to: [mac, homelab]
depends_on: []
---

# SPEC-001 — Token store

> `simple/` does not implement this spec. It pokes an application that owns its own store, and
> these requirements apply to **that** application — see
> [`simple/docs/architecture.md`](../simple/docs/architecture.md). Listing `simple` in
> `applies_to` would oblige `simple/specs/test-plan.md` to carry tests it cannot run.

## Purpose

The durable home for OAuth credentials and everything needed to reason about them: the refresh
token, the current access token, its absolute expiry, and the scopes actually granted. Every
other component reads from and writes to this one interface, so the storage backend
(SQLite, Postgres, Vault-backed) is swappable per option.

Without it, a container restart loses access and a human must re-consent. The source material's
warning is the requirement: *never store only the access token.*

## Definitions

- **`user_id`** — the stable Google account identifier (`sub` from the ID token). Not the email
  address; email addresses change and are reused.
- **`expires_at`** — an absolute UTC timestamp, derived from the token response at the moment it
  was received. Not a duration, not a local-clock assumption.
- **`granted_scopes`** — the scope set from the token *response*, which may be narrower than
  what was requested.
- **Token material** — refresh token, access token, authorization code, client secret. Never
  logged, never serialised into an error.

## Requirements

| ID | Level | Requirement |
|---|---|---|
| REQ-001-01 | MUST | Persist, per user: `user_id`, access token, refresh token, `expires_at`, `granted_scopes`, `state`, `refresh_token_issued_at`, `created_at`, `updated_at`. |
| REQ-001-02 | MUST | Persist the refresh token durably; a record holding only an access token is invalid and MUST be rejected on write. |
| REQ-001-03 | MUST | Store `expires_at` as an absolute UTC timestamp computed from the token response's expiry, never from an assumed lifetime. |
| REQ-001-04 | MUST | Key records by the Google `sub`; the email address MAY be stored as a display attribute only. |
| REQ-001-05 | MUST | Encrypt the refresh token at rest, and record which key or secret-backend version protects it so rotation can re-encrypt. |
| REQ-001-06 | MUST | Return token material only through an accessor that requires explicit unwrapping; `toString`, `JSON.stringify` and `util.inspect` on a token object MUST yield a redaction marker, not the value. |
| REQ-001-07 | MUST | Apply a token update atomically: a concurrent reader MUST see either the whole previous record or the whole new one, never a mix of old refresh token and new expiry. |
| REQ-001-08 | MUST | When a refresh response omits a refresh token, retain the stored one; MUST NOT overwrite it with null or empty. |
| REQ-001-09 | MUST | Record `granted_scopes` from the token response, replacing the stored set on full consent and merging it on incremental consent. |
| REQ-001-10 | MUST | Support marking a record `DEAD` without deleting it, preserving `user_id`, scopes and timestamps for audit. |
| REQ-001-11 | MUST | On purge, attempt server-side revocation at Google before deleting the record, and record whether revocation succeeded. |
| REQ-001-12 | MUST | Expose one interface for all backends so an option can change store technology without changing callers. |
| REQ-001-13 | MUST | Report `refresh_token_issued_at` so age-based warnings (day 6 of Testing-mode expiry) are possible. |
| REQ-001-14 | SHOULD | Provide a query for records whose `expires_at` falls before a given instant, so a sweep does not read every row. |
| REQ-001-15 | SHOULD | Survive a corrupt or unreadable record by failing that user's operations only, not the whole worker. |

## Interface sketch

```js
// lib/token-store.js

/**
 * @typedef {Object} TokenRecord
 * @property {string}   userId              Google `sub`
 * @property {string=}  email               display only
 * @property {Secret}   accessToken         redacting wrapper
 * @property {Secret}   refreshToken        redacting wrapper
 * @property {Date}     expiresAt           absolute UTC
 * @property {string[]} grantedScopes
 * @property {'VALID'|'DEAD'} state
 * @property {Date}     refreshTokenIssuedAt
 * @property {string}   keyId               encryption key / secret version
 * @property {Date}     createdAt
 * @property {Date}     updatedAt
 */

/** @returns {Promise<TokenRecord|null>} */
async function get(userId) {}

/**
 * Atomic upsert. Omitting `refreshToken` retains the stored one (REQ-001-08).
 * @throws {InvalidRecordError} when no refresh token is or would be present
 */
async function upsert(userId, fields) {}

/** @returns {Promise<void>} */
async function markDead(userId, reason) {}

/** @returns {Promise<TokenRecord[]>} */
async function listExpiringBefore(instant) {}

/** Revokes at Google, then deletes. @returns {Promise<{revoked: boolean}>} */
async function purge(userId) {}

// lib/secret.js — the redacting wrapper (REQ-001-06)
/** @returns {Secret} */
function wrap(value) {}
/** The only way out. @returns {string} */
function reveal(secret) {}
```

## Acceptance criteria

- [ ] A worker restart loses no ability to refresh: stop the container, start it, refresh succeeds.
- [ ] `pg_dump` / the SQLite file / a database backup contains no plaintext refresh token.
- [ ] Logging a whole `TokenRecord` at debug level produces no token material.
- [ ] A refresh response without a `refresh_token` leaves the stored one intact.
- [ ] Purging a user revokes at Google first, and says so.
- [ ] Swapping the backend (SQLite → Postgres, column → Vault) requires no caller changes.

## Tests

| Test ID | Covers | What it proves |
|---|---|---|
| T-001-01 | REQ-001-01 | A round-tripped record returns every field with the same values and types. |
| T-001-02 | REQ-001-02 | Writing a record with no refresh token throws `InvalidRecordError`. |
| T-001-03 | REQ-001-03 | Given a token response with a relative expiry and an injected clock, `expiresAt` equals clock + lifetime, in UTC. |
| T-001-04 | REQ-001-04 | Two accounts with the same email but different `sub` are distinct records. |
| T-001-05 | REQ-001-05 | The persisted representation of the refresh token does not contain its plaintext, and carries a `keyId`. |
| T-001-06 | REQ-001-05 | A record written under key A is readable after key B is added, and re-encryptable to B. |
| T-001-07 | REQ-001-06 | `String(record.refreshToken)`, `JSON.stringify(record)` and `util.inspect(record)` contain the redaction marker and not the value. |
| T-001-08 | REQ-001-07 | A reader interleaved with an upsert observes a self-consistent record, never a mixed one. |
| T-001-09 | REQ-001-08 | Upserting with `refreshToken` omitted leaves the stored refresh token unchanged. |
| T-001-10 | REQ-001-08 | Upserting with an explicitly empty refresh token is rejected rather than stored. |
| T-001-11 | REQ-001-09 | Full consent replaces the scope set; incremental consent yields the union, deduplicated. |
| T-001-12 | REQ-001-10 | `markDead` preserves all identity fields and the record remains readable. |
| T-001-13 | REQ-001-11 | `purge` calls the revocation endpoint before deleting, and reports revocation failure without silently skipping deletion semantics. |
| T-001-14 | REQ-001-12 | The same caller-level test suite passes against every backend implementation. |
| T-001-15 | REQ-001-13 | `refreshTokenIssuedAt` is set on consent, and unchanged by an access-token-only refresh. |
| T-001-16 | REQ-001-14 | `listExpiringBefore` returns only records inside the window, boundary exclusive of equality as specified. |
| T-001-17 | REQ-001-15 | One undecryptable record does not prevent reads of other users' records. |

## Out of scope

- The refresh decision itself — [`spec-002`](spec-002-refresh-engine.md).
- Vault paths, policies and AppRole auth — [`spec-008`](spec-008-secret-management.md) and
  [`homelab/specs/spec-302-vault-integration.md`](../homelab/specs/spec-302-vault-integration.md).
- Schema DDL and migrations for the multi-user deployment —
  [`homelab/specs/spec-303-schema-and-migrations.md`](../homelab/specs/spec-303-schema-and-migrations.md).
- Scope semantics and which scopes a feature needs — [`spec-004`](spec-004-scope-registry.md).

## References

- [`docs/token-lifecycle.md`](../docs/token-lifecycle.md)
- [`docs/security-model.md`](../docs/security-model.md)
- [ADR 0003](../docs/adr/0003-vault-openbao-for-refresh-tokens.md)
