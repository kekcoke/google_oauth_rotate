# ADR 0003 — Vault/OpenBao holds refresh tokens in `homelab/`

**Status:** Accepted

## Context

The refresh token is the long-lived credential; the source material is explicit that it must be
encrypted at rest and kept out of images and committed env files. For a multi-user deployment
that means the token store holds one password-equivalent per user.

Three storage strategies were considered: application-layer envelope encryption with a key from
the environment, encrypted secrets files (SOPS + age), and a dedicated secrets manager
(HashiCorp Vault or its fork OpenBao). The homelab already tolerates running a stateful
service, and the deployment must lift to a cloud VM unchanged.

## Decision

`homelab/` stores refresh tokens in **Vault/OpenBao** KV v2 at `secret/oauth/<user_id>/refresh`.
Postgres holds the Vault path, not the token. The worker authenticates with AppRole and a
policy that grants access to nothing else.

`simple/` and `mac/` are single-user and do not run Vault. They use an AES-256-GCM encrypted
column with the key supplied at runtime — weaker, but proportionate, and the interface the
token store exposes is identical so the backend is swappable.

## Consequences

### Good

- No plaintext refresh token in a database backup, a `pg_dump`, or a replica.
- Vault's audit device makes every refresh-token read attributable — valuable for the CASA
  assessment if verification is ever pursued.
- Key management, rotation and versioning are the secrets manager's problem rather than
  application code's.
- The same design works on a cloud VM; only the unseal mechanism changes.

### Bad

- Vault is a stateful service with an unseal ritual. **A sealed Vault means no refreshes.**
  This is the correct failure mode (fail closed) but it must alert loudly, and the unseal keys
  need a real backup story — losing them loses every stored token.
- One more container, one more upgrade path, one more thing to monitor.
- Local development needs a dev-mode Vault, so tests must not assume production sealing
  behaviour.
- `mac/` inside a Docker Desktop container cannot reach the macOS Keychain directly; the
  operator's start wrapper must export the key into the environment at `compose up` time. That
  is a documented seam, not an accident.

## Alternatives considered

- **AES-256-GCM with a key from the environment, everywhere.** Simplest, no new infra — but
  the key sits next to the ciphertext in the same process environment, and there is no audit
  trail. Kept for `simple/`/`mac/` where the threat model is one user on one host.
- **SOPS + age.** Good git hygiene, but it protects secrets at rest in a repo, not
  continuously-rotating per-user tokens; it would mean rewriting an encrypted file on every
  refresh.
- **Cloud KMS.** Ties the homelab to a cloud provider and needs network egress on the refresh
  path.

## References

- [`docs/security-model.md`](../security-model.md)
- [`homelab/specs/spec-302-vault-integration.md`](../../homelab/specs/spec-302-vault-integration.md)
- [`docs/runbooks/vault-unseal-and-rotate.md`](../runbooks/vault-unseal-and-rotate.md)
