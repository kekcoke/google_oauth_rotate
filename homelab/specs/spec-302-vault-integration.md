---
id: SPEC-302
title: Vault / OpenBao integration
status: draft
applies_to: [homelab]
depends_on: [SPEC-001, SPEC-008]
---

# SPEC-302 — Vault / OpenBao integration

## Purpose

How refresh tokens are stored in and retrieved from Vault/OpenBao, how the worker authenticates,
and what happens when Vault is unavailable. Postgres holds a path; Vault holds the credential.
The most important behaviour here is the failure behaviour: **fail closed**, and report it as one
infrastructure problem rather than as every user's token breaking at once.

## Definitions

- **KV v2** — versioned key/value secrets engine, so a bad write is recoverable.
- **AppRole** — the machine authentication method: a `role_id` in configuration plus a
  `secret_id` delivered as a runtime secret.
- **Sealed** — Vault is running but cannot decrypt its own storage; every read fails.
- **Lease** — the finite lifetime of the worker's Vault token, renewed while it works.

## Requirements

| ID | Level | Requirement |
|---|---|---|
| REQ-302-01 | MUST | Store each user's refresh token at `secret/oauth/<user_id>/refresh` in KV v2. |
| REQ-302-02 | MUST NOT | Store any token material in Postgres; the row holds the Vault path and metadata only. |
| REQ-302-03 | MUST | Authenticate with AppRole; MUST NOT use a root token in any environment other than a disposable test one. |
| REQ-302-04 | MUST | Read `secret_id` from an injected secret file or environment value, never from a committed file or an image layer. |
| REQ-302-05 | MUST | Operate under a policy limited to `create`, `update`, `read` on `secret/data/oauth/*`; a read outside that path MUST be denied. |
| REQ-302-06 | MUST | Renew the Vault lease before expiry, and treat renewal failure as backend-unavailable rather than continuing on an expired lease. |
| REQ-302-07 | MUST | Fail closed when Vault is sealed or unreachable: refuse to serve tokens; MUST NOT fall back to a cache, an environment copy, or a plaintext column. |
| REQ-302-08 | MUST | Report readiness false while the backend is unavailable, while liveness stays true. |
| REQ-302-09 | MUST | Emit exactly one alert per backend-unavailable event, not one per affected user ([spec-007](../../specs/spec-007-observability.md) REQ-007-09). |
| REQ-302-10 | MUST | Recover automatically once Vault is unsealed, without a worker restart. |
| REQ-302-11 | MUST | Write via KV v2 so the previous version is retained, and document the version-retention setting. |
| REQ-302-12 | MUST NOT | Log the Vault token, the `secret_id`, or any secret value; the path MAY be logged. |
| REQ-302-13 | MUST | Delete the Vault entry when a user is purged, after revocation at Google (with REQ-001-11). |
| REQ-302-14 | SHOULD | Enable a file audit device so every refresh-token read is attributable. |
| REQ-302-15 | SHOULD | Bound and retry Vault requests per the transient classes of [spec-009](../../specs/spec-009-error-taxonomy.md). |
| REQ-302-16 | MUST | Store the Google client secret in Vault as well, read once at start-up, and fail to start if it is absent. |

## Interface sketch

```js
// src/secrets/vault.js — implements the lib/secrets interface (spec-008)
/**
 * @throws {BackendUnavailableError} sealed, unreachable, or lease lost (REQ-302-07)
 * @throws {PermissionDeniedError}   path outside the policy (REQ-302-05)
 */
async function getRefreshToken(userId) {}      // secret/oauth/<user_id>/refresh
async function putRefreshToken(userId, secret) {}
async function deleteRefreshToken(userId) {}

/** AppRole login; schedules renewal (REQ-302-03, REQ-302-06). */
async function authenticate({ addr, roleId, secretId }) {}

/** @returns {{ reachable: boolean, sealed: boolean, leaseValidMs: number }} */
async function backendStatus() {}
```

Policy the worker runs under:

```hcl
path "secret/data/oauth/*" {
  capabilities = ["create", "update", "read"]
}
path "secret/metadata/oauth/*" {
  capabilities = ["read", "delete"]
}
# nothing else — no list on other mounts, no sys/*, no auth/*
```

## Acceptance criteria

- [ ] `psql` inspection of `oauth_tokens` shows paths, never tokens.
- [ ] The worker authenticates by AppRole with no root token present in its environment.
- [ ] A read of a path outside `secret/data/oauth/*` is denied.
- [ ] Sealing Vault makes readiness false, raises one alert, and serves no tokens.
- [ ] Unsealing restores operation without restarting the worker.
- [ ] A missing client secret prevents start-up.
- [ ] An overwritten token can be recovered from the previous KV version.
- [ ] The audit device records each refresh-token read.
- [ ] Worker logs contain paths but no secret values.

## Tests

| Test ID | Covers | What it proves |
|---|---|---|
| T-302-01 | REQ-302-01 | A stored token is readable at the specified path, and only there. |
| T-302-02 | REQ-302-02 | The Postgres row contains a path and no token material, asserted against the raw column values. |
| T-302-03 | REQ-302-03 | AppRole login succeeds; a configuration containing a root token is rejected outside test mode. |
| T-302-04 | REQ-302-04 | `secret_id` is read from the injected source; a repo-path source is refused. |
| T-302-05 | REQ-302-05 | A read outside the policy path raises `PermissionDeniedError`. |
| T-302-06 | REQ-302-06 | A lease nearing expiry is renewed; a failed renewal transitions to backend-unavailable. |
| T-302-07 | REQ-302-07 | With Vault sealed, reads throw `BackendUnavailableError` and no cached or environment value is returned. |
| T-302-08 | REQ-302-08 | While sealed, liveness is true and readiness false. |
| T-302-09 | REQ-302-09 | Ten users' operations against a sealed Vault produce exactly one alert. |
| T-302-10 | REQ-302-10 | Unsealing restores token reads with no restart. |
| T-302-11 | REQ-302-11 | An overwritten secret's previous version is retrievable. |
| T-302-12 | REQ-302-12 | Logs from authentication and reads contain the path and no secret value. |
| T-302-13 | REQ-302-13 | Purge deletes the Vault entry after revocation at Google. |
| T-302-14 | REQ-302-14 | A refresh-token read appears in the audit device output with an attributable identity. |
| T-302-15 | REQ-302-15 | A transient Vault error is retried with backoff; a permission error is not. |
| T-302-16 | REQ-302-16 | Start-up fails when the client secret is absent from Vault. |

## Out of scope

- Deploying, initialising, unsealing and backing up Vault —
  [`../../docs/runbooks/vault-unseal-and-rotate.md`](../../docs/runbooks/vault-unseal-and-rotate.md)
  and [`../docs/deploy-homelab.md`](../docs/deploy-homelab.md).
- Vault HA topology and storage backends.
- The envelope-encryption backend used by `simple/` and `mac/` —
  [spec-008](../../specs/spec-008-secret-management.md).
- Rotation procedures — the same runbook.

## References

- [ADR 0003](../../docs/adr/0003-vault-openbao-for-refresh-tokens.md)
- [`docs/security-model.md`](../../docs/security-model.md)
