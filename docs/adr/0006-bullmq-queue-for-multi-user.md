# ADR 0006 — BullMQ on Redis for the multi-user worker

**Status:** Accepted

## Context

`homelab/` implements the source material's multi-user recommendation: a queue worker, a
Postgres token table, and refresh on demand before API calls. With more than one Google
account, a bare `setInterval` sweep has problems it did not have with one user:

- One slow or failing account blocks the others in a sequential loop.
- Two concurrent jobs for the same user can both decide to refresh, producing two token
  exchanges — and depending on Google's behaviour, invalidating one of the results.
- Failures need per-job retry with backoff, not a whole-sweep retry.
- Work needs to be observable: what is queued, what failed, what is retrying.

The candidates were BullMQ (Redis), RabbitMQ, a cloud queue such as SQS, and Postgres-only
job tables (`SKIP LOCKED`).

## Decision

Use **BullMQ on Redis** for job orchestration in `homelab/`:

- One job per user per unit of work (refresh sweep, API task, `watch()` renewal)
- Per-user concurrency control so a single account is never processed twice at once
- BullMQ's built-in retry with exponential backoff, wired to the error taxonomy in
  [`spec-009`](../../specs/spec-009-error-taxonomy.md)
- Repeatable jobs for the scheduled sweeps
- Refresh correctness does **not** depend on the queue: the on-demand guard in
  [`spec-301`](../../homelab/specs/spec-301-refresh-on-demand-guard.md) checks expiry
  immediately before any API call

The per-user refresh lock is held in Redis, but the specification requires the lock to be
**advisory and expiring**: a crashed worker must not be able to block an account permanently.

## Consequences

### Good

- Per-user isolation: one dead account does not stall the rest.
- Retry, backoff, delayed jobs and repeatable schedules come for free rather than being
  hand-rolled.
- Queue depth and failure counts are natural health metrics.
- Node-native and CommonJS-compatible, matching the stack the specs target.

### Bad

- Redis becomes a required dependency and a new failure domain. If Redis is down, scheduled
  sweeps stop — hence the requirement that on-demand refresh works without the queue.
- Redis persistence must be configured deliberately; an ephemeral Redis silently loses
  repeatable job definitions on restart.
- Two stateful services (Postgres + Redis) plus Vault is a real homelab footprint.
- Distributed locking is easy to get subtly wrong. The lock must have a TTL and its expiry
  path needs a test.

## Alternatives considered

- **Postgres `SELECT … FOR UPDATE SKIP LOCKED`.** One fewer service, and transactional with the
  token write — genuinely attractive. Rejected because retries, delays and repeatable schedules
  would all be hand-built, and this project already needs Redis-class primitives for locking.
  Reconsider if the Redis footprint becomes a burden.
- **RabbitMQ.** Stronger routing semantics than needed here; heavier to operate.
- **SQS or another cloud queue.** Contradicts homelab-first deployment and adds egress on the
  refresh path.
- **No queue, just intervals.** What `mac/` does. Does not survive multiple users.

## References

- [`homelab/specs/spec-300-multi-user-queue-worker.md`](../../homelab/specs/spec-300-multi-user-queue-worker.md)
- [`docs/architecture-overview.md`](../architecture-overview.md)
