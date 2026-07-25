# Security Model

> The refresh token is the long-lived credential — treat it like a password. Encrypt it at
> rest and do not put it in Docker images or environment files committed to source control.

That line from the source material is the whole model in miniature. This document makes it
enforceable. Requirements live in
[`spec-008-secret-management.md`](../specs/spec-008-secret-management.md); the
`token-security-reviewer` agent gates changes against the rules here.

## Assets

| Asset | Lifetime | If leaked |
|---|---|---|
| **Refresh token** | Until revoked (7 days in Testing mode) | Attacker mints access tokens at will, for every granted scope, until it is revoked. Worst case in this system. |
| Client secret | Until rotated | Combined with a stolen refresh token or an intercepted code, allows token minting. Alone, limited value for installed-app flows. |
| Access token | ~1 hour | Full API access for the remaining lifetime. Bad but bounded. |
| Authorization code | Minutes, single use | Exchangeable for a refresh token — as dangerous as the refresh token if the client secret is also known. PKCE mitigates. |
| `state` / PKCE verifier | One flow | Enables CSRF or code-injection against the consent flow. |
| Pub/Sub push payloads | n/a | Mailbox metadata (`historyId`, address) — not message content, but still user data. |

Note what an attacker gets with the leaked refresh token alone under our scope set: read the
mailbox, **send mail as the user**, delete mail, and read/edit Drive files. Phishing from a
real account is the realistic consequence, not just data loss.

## Threats and mitigations

| Threat | Mitigation | Where |
|---|---|---|
| Secret committed to git | `.gitignore` covers `.env*`, `*.token`, `credentials.json`, `client_secret*.json`, Vault data; secret-scanning in CI | root, `.github/workflows/ci.yml` |
| Secret baked into an image layer | Credentials only ever injected at runtime — never `COPY`, never `--build-arg`. Image builds are tested for absence of token files | [`spec-008`](../specs/spec-008-secret-management.md) |
| Token in logs | Structured logging with a deny-list on token-shaped fields; a lint rule and a test assert no logger call receives `refresh_token`, `access_token`, `code`, or `client_secret`. Truncated prefixes are still forbidden | [`spec-007`](../specs/spec-007-observability.md) |
| Token readable in the database | `homelab/`: refresh tokens live in Vault/OpenBao, Postgres stores only a Vault reference. `simple/`/`mac/`: AES-256-GCM encrypted column, key supplied at runtime | [`spec-001`](../specs/spec-001-token-store.md), [`homelab/specs/spec-302-vault-integration.md`](../homelab/specs/spec-302-vault-integration.md) |
| Token in a crash dump or error report | Token values are wrapped so `toString`/`JSON.stringify`/`util.inspect` render a redaction marker rather than the value | [`spec-001`](../specs/spec-001-token-store.md) |
| Stolen code via redirect interception | PKCE S256 on installed-app flows; exact-match redirect URIs; single-use `state` | [`spec-003`](../specs/spec-003-consent-flow.md) |
| Over-broad access | Incremental authorization, `drive.file` over `drive`, per-call scope verification | [`docs/oauth-scopes.md`](oauth-scopes.md) |
| Forged Pub/Sub push | Verify the push request's OIDC token and audience before trusting `historyId`; treat the payload as a hint and re-fetch from Gmail | [`spec-006`](../specs/spec-006-watch-pubsub.md) |
| Compromised container reading every user's tokens | Vault policy scoped to the worker role; short leases; audit log of every read | [`spec-302`](../homelab/specs/spec-302-vault-integration.md) |
| Stale grants accumulating | Revoke server-side on user removal, not just a row delete | [`spec-001`](../specs/spec-001-token-store.md) |

## Secret storage by option

| Option | Client ID/secret | Refresh token | Encryption key |
|---|---|---|---|
| `simple/` | Runtime env from an uncommitted `.env` (mode `600`) | Encrypted column in Postgres/Redis | Runtime env, uncommitted |
| `mac/` | Runtime env sourced from an uncommitted file | Encrypted column in SQLite/Postgres on a named volume | Runtime env, uncommitted; may be exported from the macOS Keychain by the launch wrapper |
| `homelab/` | Vault KV | **Vault KV** (`secret/oauth/<user_id>/refresh`) — Postgres stores only the path | Vault-managed; no app-held DEK for refresh tokens |

`mac/` runs in a Docker Desktop container, so it cannot reach the macOS Keychain directly. If
the Keychain is the source of truth, the operator's start wrapper reads it and passes the
value in as an environment variable at `docker compose up` time; the value never lands in a
committed file. This tradeoff is recorded in
[`adr/0003-vault-openbao-for-refresh-tokens.md`](adr/0003-vault-openbao-for-refresh-tokens.md).

## Vault / OpenBao usage (`homelab/`)

- **Engine:** KV v2 at `secret/oauth/`, versioned so a bad write is recoverable.
- **Auth:** AppRole for the worker. `role_id` in config, `secret_id` delivered by the Compose
  secret mechanism; no root token in the stack, ever.
- **Policy:** the worker role gets `create`/`update`/`read` on `secret/data/oauth/*` and
  nothing else. It cannot list other paths, cannot read its own policy, cannot unseal.
- **Leases:** short-lived tokens with renewal; the worker handles renewal failure by refusing
  to serve tokens rather than degrading to a cache.
- **Audit:** file audit device enabled; every refresh-token read is attributable.
- **Unseal:** documented in [`runbooks/vault-unseal-and-rotate.md`](runbooks/vault-unseal-and-rotate.md).
  A sealed Vault means the worker cannot refresh — that is the correct failure mode, and it
  must alert rather than silently stop.

## Rotation

| What | Cadence | Procedure |
|---|---|---|
| Refresh token | Weekly in Testing mode (forced by Google); otherwise on suspicion | [`runbooks/re-consent.md`](runbooks/re-consent.md) |
| Client secret | Annually, and immediately on suspected exposure | [`runbooks/vault-unseal-and-rotate.md`](runbooks/vault-unseal-and-rotate.md) — note it invalidates existing grants, so it is a re-consent event |
| Vault AppRole `secret_id` | Quarterly | Same runbook |
| Encryption key (`simple/`/`mac/`) | Annually | Re-encrypt rows under the new key; keep the old key available until the migration completes |
| Rehearsal | Monthly | `/rotate-drill` — practise in a scratch environment so the real event is boring |

## Incident response

A suspected leak is handled by [`runbooks/incident-token-leak.md`](runbooks/incident-token-leak.md).
The first action is always **revoke at Google** (`POST https://oauth2.googleapis.com/revoke`),
because that is the only step that actually stops the attacker; cleaning up git history comes
after.

## Out of scope

- Multi-tenant authorization between users of the `homelab/` deployment beyond per-user token
  isolation — there is no end-user-facing web app in this project.
- Network security of the homelab itself (firewall, VPN, reverse-proxy TLS) is assumed
  handled by existing infrastructure; `deploy-homelab.md` states the requirements it depends on.
- Hardware security modules and hardware-backed keys.
