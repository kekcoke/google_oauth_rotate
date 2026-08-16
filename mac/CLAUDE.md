# Option 2 — `mac/`

Expiry-aware background worker, running as a Docker Desktop container on macOS. One personal
Google account, refreshed only when it actually needs it.

## What this option is

A long-running Node process on a 5-minute tick. Each tick it loads the stored token, compares
`expires_at` against `now + 10 minutes`, and refreshes only if that window is breached.
Otherwise it does nothing. It owns the token store, the consent flow and the refresh logic —
unlike [`../simple/`](../simple/), which owns none of them.

```text
every 5 minutes:
    load token
    if expires_at < now + 10 minutes: refresh()
    else: do nothing
```

## What this option is deliberately not

- **Not multi-user.** One token record. If you need several accounts, use
  [`../homelab/`](../homelab/) rather than looping here.
- **Not publicly reachable.** No ingress, so Gmail push notifications are unavailable; the
  polling fallback in [`spec-006`](../specs/spec-006-watch-pubsub.md) is used instead. Do not
  expose a laptop to the internet to get push.
- **Not a queue.** Sequential work on a timer is sufficient for one account.

## Specs

| Spec | Role here |
|---|---|
| [spec-200](specs/spec-200-expiry-aware-worker.md) | This option's own requirements: the tick, sleep/wake, single-instance |
| [spec-001](../specs/spec-001-token-store.md) | SQLite or Postgres on a named volume, encrypted refresh token |
| [spec-002](../specs/spec-002-refresh-engine.md) | The refresh decision — the heart of this option |
| [spec-003](../specs/spec-003-consent-flow.md) | Desktop-app client, loopback redirect on a fixed configured port |
| [spec-004](../specs/spec-004-scope-registry.md) | Incremental authorization, per-call coverage check |
| [spec-005](../specs/spec-005-api-adapters.md) | Gmail, Drive, Docs, Sheets adapters |
| [spec-006](../specs/spec-006-watch-pubsub.md) | Polling fallback only (REQ-006-11 … REQ-006-13) |
| [spec-007](../specs/spec-007-observability.md) | Structured logs, health report |
| [spec-008](../specs/spec-008-secret-management.md) | Encrypted column, runtime-injected key |
| [spec-009](../specs/spec-009-error-taxonomy.md) | Classification and retry |

Test plan: [`specs/test-plan.md`](specs/test-plan.md).

## Stack

- Node ≥18, CommonJS, `node:test` + c8
- `google-auth-library` for the token exchange
- SQLite (default) or Postgres, on a named Docker volume
- Docker Desktop on macOS, `restart: always`

## Local rules

- **The clock is injected. Always.** No inline `Date.now()` in decision code. This is the single
  most important rule in this option, because every interesting test is a time test.
- **A Mac sleeps.** The lid closes, the container is paused, and it resumes hours later with a
  clock jump. On resume, re-evaluate immediately rather than waiting for the next tick
  (REQ-200-04). Test this with an injected jump, never by actually sleeping.
- **This container cannot reach the macOS Keychain.** If the Keychain holds the encryption key,
  the operator's start wrapper reads it and passes it in at `compose up`. Documented seam, not
  an accident — see [`docs/deploy.md`](docs/deploy.md).
- **No token material in logs**, including on error paths and including truncated prefixes.
- **One instance only.** Two workers on one store double-refresh; guard it (REQ-200-06).
- Store state on a **named volume**, never in the container filesystem.
- Tests never call Google. The token endpoint is faked at the HTTP boundary.

## Verify a change

```sh
node --test                              # unit tests, always
docker compose build && docker compose up -d
docker compose logs -f gmail-worker      # expect "no refresh needed" most ticks
```

A healthy log is mostly boring: one decision line per tick and a refresh roughly once an hour.
Frequent refreshes mean the skew or the expiry handling is wrong.

For the real check, use `/token-health --option mac`.
