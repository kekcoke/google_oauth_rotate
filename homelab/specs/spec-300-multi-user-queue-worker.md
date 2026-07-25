---
id: SPEC-300
title: Multi-user queue worker
status: draft
applies_to: [homelab]
depends_on: [SPEC-001, SPEC-002, SPEC-009, SPEC-301, SPEC-302, SPEC-303]
---

# SPEC-300 — Multi-user queue worker

## Purpose

Job orchestration for many Google accounts: which queues exist, how per-user isolation is
enforced, how retries obey the error taxonomy, and how replicas coexist. The refresh *decision*
is [spec-002](../../specs/spec-002-refresh-engine.md); the pre-call correctness guard is
[spec-301](spec-301-refresh-on-demand-guard.md). This spec is the machinery they run inside.

## Definitions

- **Per-user lock** — an external, TTL-bounded lock keyed by `user_id`, held across the
  check-and-refresh sequence.
- **Repeatable job** — a BullMQ scheduled job (sweep, watch renewal).
- **Scheduler role** — the single process responsible for registering repeatable jobs.
- **Circuit** — per-user breaker that stops attempts after repeated transient failures.

## Requirements

| ID | Level | Requirement |
|---|---|---|
| REQ-300-01 | MUST | Process work as jobs keyed by `user_id`; no job operates on "the" token without naming a user. |
| REQ-300-02 | MUST | Allow at most one in-flight refresh per user across the whole deployment, including across replicas. |
| REQ-300-03 | MUST | Bound every lock with a TTL, so a worker that dies holding one cannot block that user permanently. |
| REQ-300-04 | MUST | Extend a held lock while legitimate work continues, so a slow-but-alive holder is not pre-empted mid-refresh. |
| REQ-300-05 | MUST | Isolate users: one user's failures, retries or circuit state MUST NOT delay or block another user's jobs. |
| REQ-300-06 | MUST | Wire queue retry to [spec-009](../../specs/spec-009-error-taxonomy.md): transient classes retry with backoff and jitter; permanent classes MUST NOT be retried by the queue. |
| REQ-300-07 | MUST NOT | Re-enqueue a job that failed with `DEAD_GRANT`, by any mechanism including BullMQ's own attempts setting. |
| REQ-300-08 | MUST | Register repeatable jobs exactly once regardless of replica count; a second replica MUST NOT duplicate schedules. |
| REQ-300-09 | MUST | Jitter repeatable job timing so replicas and users do not stampede the token endpoint. |
| REQ-300-10 | MUST | Survive Redis being unavailable without losing correctness: scheduled work stops, but on-demand refresh via [spec-301](spec-301-refresh-on-demand-guard.md) continues to work. |
| REQ-300-11 | MUST | Re-establish repeatable jobs after Redis restarts, without duplicating them. |
| REQ-300-12 | MUST | Drain gracefully on `SIGTERM`: stop accepting jobs, finish or requeue in-flight work, release locks, exit 0. |
| REQ-300-13 | MUST | Report queue depth, active count, and failure counts per queue as metrics ([spec-007](../../specs/spec-007-observability.md)). |
| REQ-300-14 | MUST | Route repeatedly failing jobs to a dead-letter destination rather than retrying indefinitely. |
| REQ-300-15 | MUST | Skip scheduled refresh work for users in state `DEAD`, and resume automatically after re-consent. |
| REQ-300-16 | SHOULD | Open a per-user circuit after a configured number of consecutive transient failures, and close it after a cooldown. |
| REQ-300-17 | SHOULD | Make job payloads carry only identifiers — never token material, never message content. |
| REQ-300-18 | MUST | Make jobs idempotent, so at-least-once delivery cannot double-apply an effect. |

## Interface sketch

```js
// src/queues.js
const QUEUES = {
  TOKEN_SWEEP:  'token-sweep',    // repeatable ~5 min, jittered
  HISTORY_SYNC: 'history-sync',   // from push receiver or polling
  WATCH_RENEW:  'watch-renew',    // repeatable hourly check
  API_TASK:     'api-task',
};

/** Registers repeatable jobs; only the scheduler-role process may call it (REQ-300-08). */
async function registerSchedules({ isScheduler }) {}

// src/locks.js — external per-user lock (REQ-300-02 … REQ-300-04)
/** @returns {Promise<Lock|null>} null when already held */
async function acquireUserLock(userId, ttlMs) {}
/** @throws {LockLostError} when the lock expired or was taken over */
async function extend(lock, ttlMs) {}
async function release(lock) {}

// src/retry-policy.js
/** Maps a taxonomy class to BullMQ options; permanent classes get attempts: 1 (REQ-300-06/07). */
function retryOptionsFor(errorClass) {}

// src/circuit.js  (REQ-300-16)
function isOpen(userId) {}
function recordFailure(userId, errorClass) {}
function recordSuccess(userId) {}
```

## Acceptance criteria

- [ ] Fifty users' sweep jobs proceed independently; one user failing every attempt does not delay
      the other forty-nine.
- [ ] Two replicas processing the same user produce exactly one token exchange.
- [ ] Killing a worker mid-refresh frees the user's lock within the TTL, and work resumes.
- [ ] A `DEAD_GRANT` job failure results in exactly one attempt, queue retries included.
- [ ] Starting two replicas produces one set of repeatable jobs.
- [ ] Stopping Redis stops sweeps but an API call still refreshes and succeeds.
- [ ] `SIGTERM` during a job leaves no lock held and no half-applied effect.
- [ ] Job payloads inspected in Redis contain no token material.

## Tests

| Test ID | Covers | What it proves |
|---|---|---|
| T-300-01 | REQ-300-01 | Every job schema requires a `user_id`; a payload without one is rejected. |
| T-300-02 | REQ-300-02 | Two workers processing one user's due token produce exactly one token exchange. |
| T-300-03 | REQ-300-02 | Fifty users processed concurrently produce fifty exchanges, not serialised behind one lock. |
| T-300-04 | REQ-300-03 | A lock whose holder vanished becomes acquirable after its TTL, on the injected clock. |
| T-300-05 | REQ-300-03 | The reclaiming worker performs the refresh; the record stays self-consistent. |
| T-300-06 | REQ-300-04 | A long-running legitimate holder extends its lock and is not pre-empted; a holder that fails to extend raises `LockLostError` and aborts before writing. |
| T-300-07 | REQ-300-05 | One user failing every attempt does not increase another user's job latency beyond a threshold. |
| T-300-08 | REQ-300-06 | Transient classes retry with jittered backoff; permanent classes are not retried. |
| T-300-09 | REQ-300-07 | A `DEAD_GRANT` failure yields exactly one attempt, asserted against BullMQ's attempt count. |
| T-300-10 | REQ-300-08 | Two replicas registering schedules produce one repeatable job per schedule. |
| T-300-11 | REQ-300-09 | Repeatable job fire times are jittered across replicas. |
| T-300-12 | REQ-300-10 | With Redis unavailable, sweeps stop and an on-demand refresh still succeeds. |
| T-300-13 | REQ-300-11 | After a Redis restart, schedules are re-established exactly once. |
| T-300-14 | REQ-300-12 | `SIGTERM` mid-job releases the lock, requeues or completes the work, and exits 0. |
| T-300-15 | REQ-300-13 | Queue depth, active count and failure count are emitted per queue. |
| T-300-16 | REQ-300-14 | A job failing past the cap lands in the dead-letter destination. |
| T-300-17 | REQ-300-15 | `DEAD` users are skipped by scheduled work, and resume after re-consent. |
| T-300-18 | REQ-300-16 | Consecutive transient failures open one user's circuit; another user's is unaffected; it closes after cooldown. |
| T-300-19 | REQ-300-17 | Job payloads read back from Redis contain no token material or message content. |
| T-300-20 | REQ-300-18 | Delivering the same job twice produces one net effect. |

## Out of scope

- The pre-call guard's semantics — [spec-301](spec-301-refresh-on-demand-guard.md).
- Vault authentication and paths — [spec-302](spec-302-vault-integration.md).
- Schema and migrations — [spec-303](spec-303-schema-and-migrations.md).
- Push receiver authentication and history semantics — [spec-006](../../specs/spec-006-watch-pubsub.md).
- Kubernetes-style scaling — deferred in [ADR 0007](../../docs/adr/0007-docker-compose-portable-deploy.md).

## References

- [ADR 0006](../../docs/adr/0006-bullmq-queue-for-multi-user.md)
- [`homelab/docs/architecture.md`](../docs/architecture.md)
