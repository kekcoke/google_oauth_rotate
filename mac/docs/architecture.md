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
|  |  health report on :8080  ->  published 127.0.0.1:8080 |  |
|  +---------------------------+---------------------------+  |
|                              |                              |
|                     +--------v--------+                     |
|                     |  token store    |  named volume       |
|                     |  SQLite/Postgres|  refresh token       |
|                     |                 |  encrypted (AES-GCM) |
|                     +--------^--------+                     |
|                              |  writes the first record     |
|  +---------------------------+---------------------------+  |
|  |  consent  (same image, profile: consent, one-shot)    |  |
|  |  :8765  ->  published 127.0.0.1:8765   [ADR 0008]     |  |
|  +-------------------------------------------------------+  |
+-------------------------------------------------------------+
        ^                       ^            |
        | TOKEN_ENC_KEY         | browser    | HTTPS
        | (Keychain wrapper)    | localhost  v
   operator shell           host browser   Google OAuth + APIs
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
| Consent helper | A **one-shot container** on the same image and volume; the browser reaches its loopback callback through a published port ([ADR 0008](../../docs/adr/0008-consent-as-a-one-shot-container.md)) |

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

The client is a **desktop-app** OAuth client with a loopback redirect. The browser runs on the
host; the store is a named volume that only a container can write. Those two facts pull in
opposite directions, and the resolution is to publish a port rather than to move the writer
([ADR 0008](../../docs/adr/0008-consent-as-a-one-shot-container.md)):

1. `scripts/consent.sh` reads `TOKEN_ENC_KEY` from the Keychain and starts the `consent` service —
   the same image as the worker, behind a Compose profile, mounting the same `tokens` volume —
   with `docker compose run --rm --service-ports consent`.
2. The container prints the authorization URL and listens on `:8765`, published to the host as
   `127.0.0.1:8765`.
3. The browser completes consent on the host and the code lands on
   `http://localhost:8765/oauth2callback`, which Docker forwards into the container. The redirect
   URI is unchanged and still matches the registered one exactly (REQ-003-14).
4. The container exchanges the code, writes the encrypted record to the volume, prints the account
   and the granted scope set, and exits. It never takes the instance guard, so it can run while
   the worker is up.
5. The worker picks the record up on its next tick.

Verify step 5 rather than assuming it — check `/health`, not just the consent helper's own output.
What this design removes is the failure it used to be verifying against: with one image, one
`DATABASE_URL` and one volume, there is no longer a second store for the record to land in.

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
