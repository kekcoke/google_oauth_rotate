---
name: vault-secrets
description: Secret handling in this project — Vault/OpenBao paths, AppRole auth, leases and fail-closed behaviour for homelab, and envelope encryption for simple and mac. Use when touching secret storage, configuration, Dockerfiles, or debugging a secrets outage.
---

# Secrets

The refresh token is the long-lived credential. Treat it like a password: encrypt it at rest, keep
it out of images and committed env files. With this project's scope set, a leaked one lets an
attacker read the mailbox, **send mail as the user**, delete mail, and read and edit Drive files
until it is revoked.

## Backends by option

| Option | Client id/secret | Refresh token | Key |
|---|---|---|---|
| `simple/` | Runtime env, uncommitted `.env` mode 600 | Encrypted column | Runtime env |
| `mac/` | Runtime env, injected by the start wrapper | Encrypted column on a named volume | Runtime env; may come from the macOS Keychain via the wrapper |
| `homelab/` | Vault KV | **Vault KV** `secret/oauth/<user_id>/refresh` | Vault-managed |

`mac/` runs in a container and **cannot read the macOS Keychain**. The operator's wrapper reads it
and injects the value at `docker compose up`. That is weaker than a secrets manager — visible in
that process's environment while it runs — and is exactly why `homelab/` uses Vault.

## Vault / OpenBao (`homelab/`)

- **Engine:** KV v2 at `secret/`, versioned so a bad write is recoverable
- **Auth:** AppRole — `role_id` in configuration, `secret_id` as an injected secret file. **No root
  token** outside a disposable test environment
- **Policy:** `create`, `update`, `read` on `secret/data/oauth/*` and nothing else
- **Leases:** renewed before expiry; a failed renewal means backend-unavailable, not "carry on"
- **Audit:** file audit device, so every refresh-token read is attributable
- **Postgres holds the path, never the token** — there is a test asserting this against raw column
  values

```hcl
path "secret/data/oauth/*"     { capabilities = ["create", "update", "read"] }
path "secret/metadata/oauth/*" { capabilities = ["read", "delete"] }
```

## Fail closed

A sealed or unreachable Vault means **refuse to serve tokens**. Never fall back to a cache, an
environment copy, or a plaintext column. Readiness goes false while liveness stays true, and it
raises **one infrastructure alert, not one per user** — a sealed Vault fails everyone
simultaneously, and N alerts hide the single cause.

Recovery is automatic on unseal, with no worker restart.

## Envelope encryption (`simple/`, `mac/`)

AES-256-GCM. The stored form carries the IV, the auth tag, and a **key identifier** — without the
key id, rotation is impossible, because you cannot tell which key a row was written under. Two keys
may be active during rotation: decrypt with either, encrypt with the new one.

## Hard rules

1. No credential in an image layer — no `COPY` of a secret, no `--build-arg`. A secret deleted in a
   later layer is still in the image.
2. No credential in git. `.env*` (except `.env.example`), `*.token`, `credentials.json`,
   `client_secret*.json`, Vault data directories.
3. No token material in logs, metric labels, span attributes, or errors — including truncated
   prefixes.
4. Fail to start on a missing or malformed secret. A worker that starts without a key and creates an
   empty store looks healthy and is useless.
5. `.env.example` lists every required variable, with no values — and a test asserts it is complete.
6. Files the app creates get mode `600`.
7. Revoke at Google **before** deleting a stored token.

## Where leaks hide

Logger calls whose argument is a whole record or config object. `JSON.stringify` and template
literals on objects containing a secret. Errors that attach the request or response. Retry code
that logs the failed request "for diagnostics". Test fixtures and snapshots. Health, debug and
metrics endpoints that dump configuration. Docker `HEALTHCHECK`/`CMD` strings visible in
`docker inspect`. Shell scripts under `set -x`.

## Rotation

| What | Cadence | Note |
|---|---|---|
| Refresh token | Weekly in Testing mode (forced) | `docs/runbooks/re-consent.md` |
| Vault `secret_id` | Quarterly | Rotation without revoking the old one is not rotation |
| Client secret | Annually, or on exposure | **Invalidates every grant** — plan the re-consents |
| Encryption key | Annually | Needs the key id in stored ciphertext |
| Drill | Monthly | `/rotate-drill` — so the real event is boring |

Back up Vault's unseal and recovery keys off-instance, in separate locations, and **test the backup
by reading it**. Losing them loses every refresh token.

## References

- `docs/security-model.md` — threat model
- `specs/spec-008-secret-management.md` — requirements
- `homelab/specs/spec-302-vault-integration.md` — Vault specifics
- `docs/runbooks/vault-unseal-and-rotate.md`, `docs/runbooks/incident-token-leak.md`
