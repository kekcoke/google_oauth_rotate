# Architecture Overview

## The problem

A Google OAuth access token is valid for roughly one hour. A naive integration refreshes it
on every request, or on a tight timer, and burns quota and latency doing it. The correct
shape is a small background worker that:

1. Stores the **refresh token** securely.
2. Checks whether the access token is near expiry.
3. Refreshes **only when needed**.
4. Stores the new access token and its expiry time.
5. Hands the valid token to the Google API client.

A cron job inside Docker can do this. A long-running worker does it better. This repository
specifies both, and a multi-user version, as three separate deliverables.

## Shared core

Regardless of option, the same four components exist. Only their implementation and
distribution differ.

```text
+---------------------------+
|  API consumer             |   Gmail / Drive / Docs / Sheets calls
|  (your app or a job)      |
+-------------+-------------+
              | needs a valid access token
              v
+---------------------------+
|  Token provider           |   getValidToken(userId, requiredScopes)
|  - is it expiring soon?   |
|  - are scopes sufficient? |
+-------------+-------------+
              | only when needed
              v
+---------------------------+
|  Refresh engine           |   refresh_token -> new access_token + expires_at
|  - single-flight per user |
|  - backoff on 429/5xx     |
|  - invalid_grant -> alert |
+-------------+-------------+
              |
              v
+---------------------------+
|  Token store              |   user_id, access_token, refresh_token,
|  (encrypted at rest)      |   expires_at, granted_scopes
+---------------------------+
```

Specified in [`spec-001-token-store.md`](../specs/spec-001-token-store.md),
[`spec-002-refresh-engine.md`](../specs/spec-002-refresh-engine.md),
[`spec-004-scope-registry.md`](../specs/spec-004-scope-registry.md).

**Never store only the access token.** If the container dies you lose access and a human has
to re-consent. The refresh token is the asset; the access token is a cache.

## Option 1 — `simple/`: Docker + cron refresh worker

```text
Docker Compose

+----------------+
| App Container  |
|                |
| Gmail API      |
| Uses tokens    |
+-------+--------+
        |
        |
+-------v--------+
| Token Worker   |
| cron every 30m |
| refresh OAuth  |
+-------+--------+
        |
        |
+-------v--------+
| Token Storage  |
| Postgres/Redis |
+----------------+
```

An Alpine container runs `crond`. Every 30 minutes a shell script POSTs to the app's
`/oauth/refresh` endpoint; the app owns the OAuth logic and the store. The worker knows
nothing about tokens — it is a scheduler with a `curl` in it.

**Choose it when** an app with a refresh endpoint already exists and you want the smallest
possible addition.
**Its weaknesses:** refreshes whether or not it needs to, a fixed 30-minute grid can still
miss an expiry if the app is down at the wrong moment, and failures are only visible in a log
file. Detail in [`simple/docs/architecture.md`](../simple/docs/architecture.md).

## Option 2 — `mac/`: expiry-aware background worker

Instead of cron guessing:

```text
every 5 minutes:
    load token
    if expires_at < now + 10 minutes:
        refresh()
    else:
        do nothing
```

A long-running Node process on a 5-minute tick inside a Docker Desktop container with
`restart: always`. It reads the stored expiry and refreshes only inside the skew window.

**Benefits:** no unnecessary refresh calls, handles restarts, easier logging, easier scaling.
**Choose it when** you want one dependable personal worker on your Mac.
**Its weaknesses:** single-host, no ingress for Pub/Sub push (polling fallback instead), and
tied to Docker Desktop running. Detail in [`mac/docs/architecture.md`](../mac/docs/architecture.md).

## Option 3 — `homelab/`: multi-user worker, homelab → cloud

Option 3 in the source material is "local Docker + cloud deployment friendly": persistent
volumes, env-injected client credentials, a real `oauth_tokens` table. This repository takes
that and adopts the source's **production, multi-user** recommendation:

- Queue worker (BullMQ on Redis) rather than a bare interval
- Postgres token table keyed by `user_id`
- **Refresh on demand, immediately before the API call** — not only on a timer
- Per-user advisory locking so two concurrent jobs cannot double-refresh one account
- Refresh tokens held in Vault/OpenBao, never as a plaintext Postgres column

```text
API request
   |
   v
Need Google API?
   |
   v
Check token expiry
   |
 +--+---+
 |      |
valid  expired
 |      |
call   refresh (locked, single-flight)
   |      |
   +--+---+
      v
   Google API
```

The timer-based sweep still exists as a background safety net, but correctness does not
depend on it: any call path can find a stale token and fix it before use.

**Choose it when** more than one Google account is involved, or the stack must move to a
cloud VM without a rewrite. The same Compose file runs in both places; only secret sourcing
and TLS termination differ. Detail in
[`homelab/docs/architecture.md`](../homelab/docs/architecture.md).

## Comparison

| | `simple/` | `mac/` | `homelab/` |
|---|---|---|---|
| Trigger | cron, every 30 min | interval, every 5 min | on-demand + queue + sweep |
| Refresh decision | unconditional | `expires_at` vs skew window | `expires_at` vs skew window |
| Users | 1 | 1 | many |
| Store | Postgres/Redis | SQLite/Postgres | Postgres |
| Refresh token at rest | encrypted column | encrypted column | Vault/OpenBao |
| Concurrency safety | none needed | none needed | per-user lock, single-flight |
| Pub/Sub push | no (poll) | no (poll) | yes, HTTPS receiver |
| Failure visibility | log file | structured logs + health | logs, metrics, alerts |
| Ops burden | lowest | low | real |

## What every option must handle

These are requirements, not nice-to-haves. Each is traced to a spec:

- **Near-expiry, not post-expiry, refresh** — a token that expires mid-request is an outage.
  [`spec-002`](../specs/spec-002-refresh-engine.md)
- **`invalid_grant`** — the refresh token is dead (7-day Testing-mode expiry, revocation,
  password change, or scope change). Detect, alert, do not retry in a loop.
  [`spec-009`](../specs/spec-009-error-taxonomy.md)
- **Insufficient scope** — incremental authorization means a stored token may not carry the
  scope a new feature needs. Check before calling.
  [`spec-004`](../specs/spec-004-scope-registry.md)
- **Rate limiting** — `429` with `Retry-After`, exponential backoff with jitter.
  [`spec-009`](../specs/spec-009-error-taxonomy.md)
- **Restart safety** — state lives in the store, never only in memory.
  [`spec-001`](../specs/spec-001-token-store.md)
- **Secret hygiene** — nothing sensitive in images, logs, or committed env files.
  [`spec-008`](../specs/spec-008-secret-management.md)

## Source material

Distilled from `gmail oauth.pdf` (see `docs/source/` once committed). Where this repository
departs from it, an ADR records why — see [`adr/`](adr/).
