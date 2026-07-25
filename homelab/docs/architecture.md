# `homelab/` — Architecture

```text
                    Google APIs / OAuth token endpoint
                              ^        ^
                              |        |
        +---------------------+        +--------------------+
        |                                                   |
+-------+---------------------------------------------------+-------+
|  worker (Node, BullMQ consumer + repeatable schedules)            |
|                                                                   |
|   +-----------------------------+   +--------------------------+  |
|   |  on-demand guard (spec-301) |   |  sweep job (safety net)  |  |
|   |  before EVERY API call:     |   |  repeatable, jittered    |  |
|   |   lock(user) -> check exp   |   |  refreshes near-expiry   |  |
|   |   -> refresh if needed      |   +--------------------------+  |
|   +-----------------------------+                                 |
|                                                                   |
|   +--------------+  +--------------+  +------------------------+  |
|   | watch renew  |  | history sync |  | adapters gmail/drive/  |  |
|   | (7-day clock)|  | (historyId)  |  | docs/sheets            |  |
|   +--------------+  +--------------+  +------------------------+  |
+----+---------------------+----------------------+-----------------+
     |                     |                      |
     v                     v                      v
+----------+        +-------------+       +-----------------+
|  Redis   |        |  Postgres   |       |  Vault/OpenBao  |
|  BullMQ  |        |  oauth_     |       |  secret/oauth/  |
|  locks   |        |  tokens     |       |  <user>/refresh |
+----------+        +-------------+       +-----------------+
                          ^                        ^
                          | path only,              | refresh tokens,
                          | never a token           | client secret
                          |
+-------------------------+-----------------------------------------+
|  push receiver (HTTPS)  <-- Pub/Sub push <-- Gmail watch()        |
|  verifies OIDC token, enqueues history-sync job                   |
+-------------------------------------------------------------------+
```

## Components

| Component | Responsibility | Spec |
|---|---|---|
| On-demand guard | Before any Google call: lock the user, check expiry, refresh if needed | [301](../specs/spec-301-refresh-on-demand-guard.md) |
| Queue worker | Consumes jobs, per-user concurrency, retry with backoff | [300](../specs/spec-300-multi-user-queue-worker.md) |
| Sweep job | Repeatable; refreshes near-expiry tokens ahead of use | [300](../specs/spec-300-multi-user-queue-worker.md), [002](../../specs/spec-002-refresh-engine.md) |
| Watch renewal job | Repeatable; re-registers `watch()` inside 48 h of expiry | [006](../../specs/spec-006-watch-pubsub.md) |
| Push receiver | Verifies the OIDC token, enqueues a history sync | [006](../../specs/spec-006-watch-pubsub.md) |
| Token store | Postgres row + Vault reference | [001](../../specs/spec-001-token-store.md), [303](../specs/spec-303-schema-and-migrations.md) |
| Secret backend | Vault/OpenBao, AppRole, fail-closed | [302](../specs/spec-302-vault-integration.md) |

## Refresh on demand — the correctness boundary

```text
API request                      (the source material's flow, with locking added)
   |
   v
Need Google API?
   |
   v
acquire per-user lock (TTL)
   |
   v
Check token expiry
   |
 +--+-------+
 |          |
valid     near-expiry / expired
 |          |
 |          v
 |       refresh  --invalid_grant--> mark DEAD, alert, fail this call
 |          |          (never retried)
 |          v
 +------> release lock
   |
   v
Google API call
```

Why the lock wraps the check as well as the refresh: two jobs for one user that both read "not
expired" a millisecond before expiry will both proceed, and one of them will fail mid-call.
Checking inside the lock makes the decision and the action atomic per user.

Why the lock has a TTL: a worker that dies holding it must not block that account forever. The
TTL expiry path is tested (T-300-05).

## Job topology

| Queue / job | Trigger | Concurrency | Retry |
|---|---|---|---|
| `token-sweep` | Repeatable, ~5 min, jittered | 1 per user | Transient classes only |
| `history-sync` | Push receiver, or polling | 1 per user | Transient classes only |
| `watch-renew` | Repeatable, hourly check | 1 per user | Transient classes only |
| `api-task` | Application work | Configurable, still 1 refresh per user | Per taxonomy |

Every queue's retry is wired to [spec-009](../../specs/spec-009-error-taxonomy.md). BullMQ's own
retry must not resurrect a `DEAD_GRANT` failure — the class travels with the job, and this is
easy to get wrong.

## Two independent 7-day clocks

| Clock | Expires | Symptom when it lapses | Check |
|---|---|---|---|
| Refresh token (Testing mode) | 7 days after issue | `invalid_grant` on every refresh for that user | Daily token-health, day-6 warning |
| Gmail `watch()` | ~7 days after registration | **Silence** — no pushes, no errors | Hourly renewal check, plus a silence alarm |

Conflating them wastes a day of debugging. A user can have a perfectly healthy token and a dead
watch, or a live watch and a dead token.

## Multi-user consequences

- Different users hold **different scope sets**; coverage is evaluated per user.
- One broken account must not consume the retry budget of the rest — hence per-user circuit
  breaking (REQ-009-12).
- A sealed Vault fails every user simultaneously; that is **one** infrastructure alert, not N
  token alerts (REQ-007-09).
- Replicas are possible because locking is per-user and external, but repeatable jobs must not be
  registered twice — see [spec-300](../specs/spec-300-multi-user-queue-worker.md).

## Failure modes

| Failure | Blast radius | Behaviour |
|---|---|---|
| Vault sealed | All users | Fail closed, readiness false, one alert |
| Redis down | Scheduled sweeps and queued work | On-demand refresh still works — the reason the guard exists |
| Postgres down | All users | Readiness false, alert; no token operations |
| One user's grant dead | That user | `DEAD`, alert, no retries, others unaffected |
| Worker crash holding a lock | That user, briefly | Lock TTL expires and work resumes |
| Push receiver unreachable | Real-time updates | Pub/Sub backlog grows (the signal), history sync catches up on the next poll |
| `watch()` lapsed | Real-time updates for that user | Silence alarm; recover from the stored `historyId` |
| Two replicas both registering repeatable jobs | Duplicate scheduled work | Prevented by the single-scheduler requirement |

## Portability to a cloud VM

Same Compose file. What differs is configuration, not architecture: where Vault's unseal keys
come from, who terminates TLS in front of the push receiver, and how volumes are backed up. See
[`deploy-homelab.md`](deploy-homelab.md) and [`deploy-cloud.md`](deploy-cloud.md), and
[ADR 0007](../../docs/adr/0007-docker-compose-portable-deploy.md) for why Kubernetes was deferred.
