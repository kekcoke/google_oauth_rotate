---
id: SPEC-200
title: Expiry-aware worker (macOS / Docker Desktop)
status: draft
applies_to: [mac]
depends_on: [SPEC-001, SPEC-002, SPEC-003, SPEC-007, SPEC-008, SPEC-009]
---

# SPEC-200 — Expiry-aware worker (macOS / Docker Desktop)

## Purpose

The process that hosts the refresh engine on a personal Mac: how it ticks, how it survives the
host sleeping, how it refuses to run twice, and how it starts and stops cleanly. The refresh
*decision* is [spec-002](../../specs/spec-002-refresh-engine.md); this spec is everything around
it that is specific to being a long-running container on a laptop.

## Definitions

- **Tick** — one scheduled evaluation pass. Default interval **5 minutes**.
- **Resume event** — wall-clock elapsed time between ticks exceeding the expected interval by
  more than the resume threshold (default 2 minutes), indicating the host slept.
- **Single-instance guard** — the mechanism preventing two workers from operating one store.

## Requirements

| ID | Level | Requirement |
|---|---|---|
| REQ-200-01 | MUST | Evaluate every token on a fixed interval, default 5 minutes, configurable. |
| REQ-200-02 | MUST | Evaluate once at start-up, before the first interval elapses. |
| REQ-200-03 | MUST NOT | Allow two evaluations to overlap; a slow tick delays the next rather than running concurrently. |
| REQ-200-04 | MUST | Detect a resume event and evaluate immediately, without waiting for the next tick. |
| REQ-200-05 | MUST | Derive all timing from the injected clock; MUST NOT infer elapsed time from tick counts. |
| REQ-200-06 | MUST | Refuse to start when another instance holds the store, exiting non-zero with a clear message. |
| REQ-200-07 | MUST | Fail to start when a required secret or the store is unavailable, rather than starting a degraded worker or creating an empty store. |
| REQ-200-08 | MUST | Shut down gracefully on `SIGTERM`: finish or abandon an in-flight refresh without leaving a partially written record, release the instance guard, exit 0. |
| REQ-200-09 | MUST | Persist all state to the mounted volume; MUST NOT keep authoritative state in memory only. |
| REQ-200-10 | MUST | Expose the health report of [spec-007](../../specs/spec-007-observability.md) over HTTP on a configurable port, bound to localhost only. |
| REQ-200-11 | MUST | Log one decision line per tick, including whether a refresh was performed and the seconds remaining until expiry. |
| REQ-200-12 | MUST | Treat an absent token record as a reportable state (not consented), not an error loop. |
| REQ-200-13 | MUST NOT | Retry a `DEAD` token on subsequent ticks; alert at most once per configured interval instead. |
| REQ-200-14 | MUST | Run the polling fallback for mailbox changes ([spec-006](../../specs/spec-006-watch-pubsub.md) REQ-006-11 … REQ-006-13); MUST NOT attempt `watch()` registration. |
| REQ-200-15 | MUST | Accept the encryption key by runtime injection, and document that the container cannot read the macOS Keychain itself. |
| REQ-200-16 | SHOULD | Jitter tick timing so behaviour is not synchronised with any other scheduled work. |
| REQ-200-17 | SHOULD | Keep idle resource use negligible — an idle tick does no I/O beyond one store read and one log line. |
| REQ-200-18 | SHOULD | Operate in UTC internally, formatting local time only for human-facing output. |

## Interface sketch

```js
// src/worker.js
/**
 * @param {Object} deps
 * @param {() => Date}     deps.now
 * @param {number}         deps.tickMs           default 300000
 * @param {number}         deps.resumeThresholdMs default 120000
 * @param {RefreshEngine}  deps.engine
 * @param {TokenStore}     deps.store
 * @param {InstanceGuard}  deps.guard
 * @param {Logger}         deps.logger
 */
function createWorker(deps) {
  return {
    /** Acquires the guard, evaluates once, then schedules ticks. @throws {InstanceHeldError} */
    async start() {},
    /** One evaluation pass over all records. */
    async tick() {},
    /** Graceful shutdown (REQ-200-08). */
    async stop() {},
  };
}

/** True when elapsed exceeds expected by more than the threshold (REQ-200-04). */
function isResumeEvent(expectedMs, actualMs, thresholdMs) {}

// src/instance-guard.js — store-backed lock with a stale-holder timeout
async function acquire(storePath, ttlMs) {}
async function release() {}
```

## Acceptance criteria

- [ ] Over a simulated day, one refresh occurs per token lifetime — not one per tick.
- [ ] An injected two-hour clock jump triggers immediate re-evaluation and one refresh.
- [ ] Starting a second worker against the same store exits non-zero without touching the record.
- [ ] `SIGTERM` mid-refresh leaves a self-consistent record and releases the guard.
- [ ] Starting with `TOKEN_ENC_KEY` unset exits non-zero and creates no store file.
- [ ] `docker compose restart` loses no state and resumes on the next tick.
- [ ] The health endpoint is unreachable from another host on the network.
- [ ] No test sleeps; every timing behaviour is driven by the injected clock.

## Tests

| Test ID | Covers | What it proves |
|---|---|---|
| T-200-01 | REQ-200-01 | Advancing the injected clock by the interval triggers exactly one evaluation. |
| T-200-02 | REQ-200-02 | `start()` evaluates once before any interval has elapsed. |
| T-200-03 | REQ-200-03 | A tick that outlasts the interval delays the next tick instead of overlapping. |
| T-200-04 | REQ-200-04 | An elapsed gap beyond the resume threshold triggers immediate evaluation; a gap under it does not. |
| T-200-05 | REQ-200-05 | Behaviour depends only on the injected clock; no wall-clock reads in decision paths. |
| T-200-06 | REQ-200-06 | A second instance against the same store throws `InstanceHeldError` and exits non-zero. |
| T-200-07 | REQ-200-06 | A stale guard whose holder died is reclaimed after its TTL. |
| T-200-08 | REQ-200-07 | Missing key, unreachable store, and missing volume each prevent start-up; no store file is created. |
| T-200-09 | REQ-200-08 | `SIGTERM` during a refresh yields a self-consistent record, a released guard, and exit 0. |
| T-200-10 | REQ-200-09 | Restarting the worker preserves state and does not re-refresh a still-valid token. |
| T-200-11 | REQ-200-10 | The health endpoint serves the specified fields and binds to localhost only. |
| T-200-12 | REQ-200-11 | Every tick emits one decision line with the refresh flag and seconds remaining. |
| T-200-13 | REQ-200-12 | With no record, the worker reports "not consented" and keeps ticking without error. |
| T-200-14 | REQ-200-13 | A `DEAD` record produces no exchange attempts across many ticks, and at most one alert per interval. |
| T-200-15 | REQ-200-14 | Polling mode runs and no `watch()` call is attempted. |
| T-200-16 | REQ-200-15 | The key is read from the environment; absent it, start-up fails (with T-200-08). |
| T-200-17 | REQ-200-16 | Two workers with jitter enabled do not tick in lockstep. |
| T-200-18 | REQ-200-17 | An idle tick performs one store read and emits one log line — no exchange, no extra I/O. |
| T-200-19 | REQ-200-18 | Stored and compared times are UTC; only human-facing output is localised. |

## Out of scope

- The refresh decision and the exchange itself — [spec-002](../../specs/spec-002-refresh-engine.md).
- The consent flow, which runs on the host — [spec-003](../../specs/spec-003-consent-flow.md) and
  [`../docs/deploy.md`](../docs/deploy.md).
- Gmail push notifications; unavailable without ingress.
- Multi-user scheduling, queues and distributed locking — [`../../homelab/`](../../homelab/).
- Docker Desktop's own availability; a stopped Docker Desktop cannot be detected from inside a
  container that is not running.

## References

- [`mac/docs/architecture.md`](../docs/architecture.md)
- [ADR 0002](../../docs/adr/0002-expiry-aware-refresh-over-fixed-cron.md)
