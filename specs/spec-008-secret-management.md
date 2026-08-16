---
id: SPEC-008
title: Secret management
status: accepted
applies_to: [simple, mac, homelab]
depends_on: [SPEC-001]
---

# SPEC-008 — Secret management

> **Partial by option.** REQ-008-09, REQ-008-10, REQ-008-13 and REQ-008-14 are Vault-specific and
> apply to `homelab/` only. `simple/` owns no token store, so the encryption requirements
> (REQ-008-04, REQ-008-06, REQ-008-12) apply to the application it pokes rather than to it. Each
> option's plan names the test IDs it excludes.

## Purpose

How credentials enter the running system and how they are protected while stored. The source
material states the rule this spec enforces: *the refresh token is the long-lived credential —
treat it like a password; encrypt it at rest and do not put it in Docker images or environment
files committed to source control.*

This spec is the machine-checkable version of [`docs/security-model.md`](../docs/security-model.md).

## Definitions

- **Runtime injection** — a secret reaching a process as an environment variable, mounted file,
  or secret-manager read at start-up, from a source that is not in the image and not in git.
- **Envelope encryption** — the token is encrypted with a data key; the key is supplied at
  runtime and identified in the ciphertext so it can be rotated.
- **Secret backend** — Vault/OpenBao in `homelab/`; an encrypted column plus a runtime key in
  `simple/` and `mac/`.

## Requirements

| ID | Level | Requirement |
|---|---|---|
| REQ-008-01 | MUST NOT | Include any credential in a built image layer — no `COPY` of a secret file, no `ARG`/`--build-arg`, no secret baked into a base image. |
| REQ-008-02 | MUST NOT | Commit any credential to version control; `.gitignore` MUST cover `.env*` (except `.env.example`), `*.token`, `credentials.json`, `client_secret*.json`, and secret-backend data directories. |
| REQ-008-03 | MUST | Deliver client id, client secret and encryption key by runtime injection only. |
| REQ-008-04 | MUST | Encrypt refresh tokens at rest with AES-256-GCM (or the secret backend's equivalent), storing an authentication tag and a key identifier. |
| REQ-008-05 | MUST | Fail to start when a required secret is absent or malformed; MUST NOT start in a degraded mode that appears to work. |
| REQ-008-06 | MUST | Support two active encryption keys during rotation: decrypt with either, encrypt with the new one. |
| REQ-008-07 | MUST | Ship a `.env.example` listing every required variable by name, with no real values. |
| REQ-008-08 | MUST | Set restrictive permissions on any secret file it creates (`600` for files, `700` for directories). |
| REQ-008-09 | MUST | In `homelab/`, read refresh tokens from Vault at `secret/oauth/<user_id>/refresh`; Postgres MUST store only the path, never the token. |
| REQ-008-10 | MUST | In `homelab/`, authenticate to Vault with AppRole under a policy limited to `secret/data/oauth/*`; MUST NOT use a root token. |
| REQ-008-11 | MUST | Fail closed when the secret backend is unavailable: refuse to serve tokens rather than falling back to a cache or a plaintext copy. |
| REQ-008-12 | MUST | Revoke the credential at Google before deleting a stored token during purge (with REQ-001-11). |
| REQ-008-13 | SHOULD | Enable the secret backend's audit device so every refresh-token read is attributable. |
| REQ-008-14 | SHOULD | Renew Vault leases before expiry, and treat renewal failure as backend-unavailable rather than continuing on an expired lease. |
| REQ-008-15 | MUST | Provide a build-time check that the produced image contains none of the secret filename patterns from REQ-008-02. |

## Interface sketch

```js
// lib/secrets/index.js — one interface, three backends (REQ-008-09, REQ-008-11)
/**
 * @throws {ConfigError}            required secret missing at start-up (REQ-008-05)
 * @throws {BackendUnavailableError} sealed/unreachable backend (REQ-008-11)
 */
async function getRefreshToken(userId) {}
async function putRefreshToken(userId, secret) {}

// lib/secrets/envelope.js  (simple/, mac/)
/** @returns {{ciphertext: string, iv: string, tag: string, keyId: string}} */
function encrypt(plaintext, key, keyId) {}
/** Selects the key by `keyId`, supporting two active keys (REQ-008-06). */
function decrypt(envelope, keyring) {}

// lib/config.js
/** Validates every required variable at start-up; throws, never defaults a secret. */
function loadConfig(env) {}
```

Required environment variables, documented in `.env.example`:

```text
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REDIRECT_URI=
OAUTH_PUBLISHING_STATUS=  # testing | production | internal — drives the 7-day
                          # refresh-token expiry warnings (REQ-002-13, REQ-007-08).
                          # Not a secret, but required: the code cannot discover it.
TOKEN_ENC_KEY=            # simple/, mac/ — 32 bytes, base64
TOKEN_ENC_KEY_ID=         # identifies the key in stored ciphertext
DATABASE_URL=
VAULT_ADDR=               # homelab/
VAULT_ROLE_ID=            # homelab/
VAULT_SECRET_ID_FILE=     # homelab/ — path to an injected secret, not the value
```

## Acceptance criteria

- [ ] `docker history` and a filesystem scan of the built image find no secret file or value.
- [ ] `git ls-files` matches none of the ignored secret patterns; a deliberate attempt to add one
      is refused by CI.
- [ ] Starting with a missing `GOOGLE_CLIENT_SECRET` exits non-zero with a clear message.
- [ ] A token encrypted under key A is readable after rotating to key B, and re-encryptable.
- [ ] A sealed Vault causes readiness failure and an alert, and no token is served.
- [ ] `.env.example` lists every variable `loadConfig` requires — verified by a test, not by eye.

## Tests

| Test ID | Covers | What it proves |
|---|---|---|
| T-008-01 | REQ-008-01 | The built image contains no file matching the secret patterns, at any layer. |
| T-008-02 | REQ-008-02 | The ignore rules match each forbidden pattern; a staged secret file is detected. |
| T-008-03 | REQ-008-03 | Configuration is read from the environment only; no secret is read from a repo path. |
| T-008-04 | REQ-008-04 | Ciphertext round-trips, includes an auth tag and `keyId`, and a tampered ciphertext fails to decrypt. |
| T-008-05 | REQ-008-05 | Each missing or malformed required variable causes a start-up failure naming it. |
| T-008-06 | REQ-008-06 | With two keys loaded, records under either decrypt, and new writes use the new key. |
| T-008-07 | REQ-008-07 | Every variable required by `loadConfig` appears in `.env.example`, and none has a value. |
| T-008-08 | REQ-008-08 | A secret file created by the app has mode `600`. |
| T-008-09 | REQ-008-09 | The Postgres row contains a Vault path and no token material. |
| T-008-10 | REQ-008-10 | The worker authenticates by AppRole; a read outside `secret/data/oauth/*` is denied. |
| T-008-11 | REQ-008-11 | With the backend sealed, token reads throw `BackendUnavailableError` and no cached value is returned. |
| T-008-12 | REQ-008-12 | Purge calls revocation before deletion. |
| T-008-13 | REQ-008-13 | A refresh-token read appears in the audit device output. |
| T-008-14 | REQ-008-14 | A failed lease renewal transitions to backend-unavailable rather than continuing. |
| T-008-15 | REQ-008-15 | The image check fails a deliberately poisoned build. |

## Out of scope

- Threat model narrative and rotation procedures — [`docs/security-model.md`](../docs/security-model.md)
  and [`docs/runbooks/vault-unseal-and-rotate.md`](../docs/runbooks/vault-unseal-and-rotate.md).
- Vault deployment topology, HA and unsealing —
  [`homelab/specs/spec-302-vault-integration.md`](../homelab/specs/spec-302-vault-integration.md).
- Host and network hardening.

## References

- [ADR 0003](../docs/adr/0003-vault-openbao-for-refresh-tokens.md)
- [`docs/security-model.md`](../docs/security-model.md)
