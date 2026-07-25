# `mac/` — Architecture

```text
macOS host
+-------------------------------------------------------------+
|  Docker Desktop                                             |
|                                                             |
|  +-------------------------------------------------------+  |
|  |  gmail-worker  (Node, restart: always)                |  |
|  |                                                       |  |
|  |  scheduler ──5 min──> refresh engine                  |  |
|  |      |                    |                           |  |
|  |      |                    +--> needsRefresh(exp,now)? |  |
|  |      |                            no  -> log, done    |  |
|  |      |                            yes -> exchange     |  |
|  |      |                                                |  |
|  |  wake detector ──clock jump──> immediate re-evaluate   |  |
|  |                                                       |  |
|  |  adapters: gmail | drive | docs | sheets              |  |
|  |  health report on :8080/health                        |  |
|  +---------------------------+---------------------------+  |
|                              |                              |
|                     +--------v--------+                     |
|                     |  token store    |  named volume       |
|                     |  SQLite/Postgres|  refresh token       |
|                     |                 |  encrypted (AES-GCM) |
|                     +-----------------+                     |
+-------------------------------------------------------------+
        ^                                    |
        | TOKEN_ENC_KEY at `compose up`      | HTTPS
        | (operator wrapper, e.g. Keychain)  v
   operator shell                      Google OAuth + APIs
```

## Components

| Component | Responsibility |
|---|---|
| Scheduler | Fires the evaluation on a 5-minute tick, with jitter; no overlap |
| Wake detector | Notices a clock jump (host sleep) and forces immediate re-evaluation |
| Refresh engine | `needsRefresh(expiresAt, now, skew)` and the token exchange ([spec-002](../../specs/spec-002-refresh-engine.md)) |
| Token store | SQLite or Postgres on a named volume, refresh token encrypted ([spec-001](../../specs/spec-001-token-store.md)) |
| Adapters | Gmail, Drive, Docs, Sheets, each asserting scope coverage first ([spec-005](../../specs/spec-005-api-adapters.md)) |
| Health report | Token state, seconds to expiry, refresh-token age ([spec-007](../../specs/spec-007-observability.md)) |
| Consent helper | Runs on the **host**, not in the container — loopback redirect needs a browser |

## The tick

```text
tick (every 5 min, jittered)
   |
   v
load token record
   |
   +-- no record ----------> log "not consented", surface in health, done
   +-- state DEAD ---------> log, alert (once per interval), done  [no retry]
   |
   v
needsRefresh(expiresAt, now, skew = 10 min)?
   |
   +-- no ---> log decision + seconds remaining, done   (the common case)
   |
   +-- yes --> acquire in-process single-flight
                  |
                  v
               token exchange
                  |
                  +-- success -------> persist atomically, log, metric
                  +-- invalid_grant -> mark DEAD, alert, STOP
                  +-- 429/5xx/net ---> backoff + jitter, retry to cap, then alert
```

Most ticks do nothing. That is the design working: roughly one exchange per token lifetime
rather than 48 a day.

## Sleep and wake — the interesting part

A Mac is not a server. The lid closes, Docker Desktop pauses the container, and it resumes hours
later. Two consequences:

1. **The token is already expired on resume.** Waiting up to five minutes for the next tick means
   requests fail in the meantime. So a detected jump forces immediate re-evaluation
   (REQ-200-04).
2. **Timers do not fire while paused.** A `setInterval` does not "catch up"; the elapsed time is
   simply gone. Any logic that counts ticks rather than reading the clock is wrong.

The wake detector compares wall-clock elapsed against expected elapsed each tick, and treats a
discrepancy beyond a threshold as a resume event. Tested with an injected clock — never by
sleeping a test.

## Consent, and why it straddles the boundary

The client is a **desktop-app** OAuth client with a loopback redirect. The container has no
browser and the loopback redirect must land where the browser is, so:

1. The operator runs the consent helper on the host.
2. The browser completes consent and the code lands on `http://localhost:<port>/oauth2callback`.
3. The helper exchanges the code and writes the record to the store — the same volume the
   container uses.
4. The container picks it up on its next tick.

Verify step 4 rather than assuming it. A token written to the wrong path looks identical to a
token that was never written.

## Failure modes

| Failure | Detection | Response |
|---|---|---|
| Refresh token dead (weekly, Testing mode) | `invalid_grant` → `DEAD`; day-6 age warning arrives first | Re-consent ([runbook](../../docs/runbooks/re-consent.md)) |
| Docker Desktop not running | No logs, no health endpoint | Host-level check; this option cannot self-report it |
| Host slept through an expiry | Wake detector | Immediate re-evaluation on resume |
| Two workers on one store | Single-instance guard (REQ-200-06) | Second instance refuses to start |
| Encryption key missing at start | Start-up validation | Refuse to start, non-zero exit, clear message |
| Volume not mounted | Store unreachable | Readiness false, alert; do **not** create a fresh empty store |
| Clock badly wrong on the host | Expiry comparisons skewed | Out of scope; noted as an assumption |

## Limits of this option

Single account, single host, no ingress, no queue. When any of those stops being true —
a second Google account, a need for real-time push, or a requirement that the automation survive
the laptop being closed — move to [`../../homelab/`](../../homelab/) rather than growing this one.
