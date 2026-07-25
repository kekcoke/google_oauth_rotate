---
id: SPEC-301
title: Refresh-on-demand guard
status: draft
applies_to: [homelab]
depends_on: [SPEC-002, SPEC-004, SPEC-009, SPEC-300]
---

# SPEC-301 — Refresh-on-demand guard

## Purpose

The correctness boundary of this option. Every Google API call passes through a guard that, under
a per-user lock, checks expiry and scope coverage and refreshes if needed — **immediately before
the call**. The scheduled sweep keeps tokens warm and surfaces problems early, but the system
does not depend on it. If Redis is down, if the sweep is paused, if the worker started thirty
seconds ago, an API call still gets a valid token.

This is the source material's production recommendation for multiple users: *refresh on demand
before API calls*, rather than trusting a timer.

## Definitions

- **Guard** — the wrapper every adapter call goes through.
- **Check-and-refresh** — reading expiry and refreshing, as one atomic sequence per user.
- **Bypass** — any code path reaching Google without the guard. There are none; that is the point.

## Requirements

| ID | Level | Requirement |
|---|---|---|
| REQ-301-01 | MUST | Perform check-and-refresh immediately before every Google API call. |
| REQ-301-02 | MUST | Hold the per-user lock across **both** the expiry check and the refresh, so the decision and the action are atomic per user. |
| REQ-301-03 | MUST | Wait for an in-flight refresh by another holder and then use its result, rather than starting a second refresh or failing. |
| REQ-301-04 | MUST | Bound the wait for another holder's refresh, and classify a timeout as transient. |
| REQ-301-05 | MUST | Assert scope coverage ([spec-004](../../specs/spec-004-scope-registry.md)) inside the guard, before the call and before any refresh. |
| REQ-301-06 | MUST | Work when the queue is unavailable; the guard MUST NOT depend on BullMQ. |
| REQ-301-07 | MUST | Fail the call — never proceed with an expired or absent token — when refresh is impossible. |
| REQ-301-08 | MUST NOT | Retry `invalid_grant`; mark `DEAD`, alert, and fail the call. |
| REQ-301-09 | MUST | Make it structurally impossible for an adapter to reach Google without the guard: adapters receive their token only through it, and a test asserts no bypass exists. |
| REQ-301-10 | MUST | Add no refresh when the token is comfortably valid — the guard's common path is one expiry comparison and no I/O beyond what it already needed. |
| REQ-301-11 | MUST | Release the lock on every exit path, including thrown errors and timeouts. |
| REQ-301-12 | MUST | Propagate a correlation id so the guard's decision and the resulting API call share one trace ([spec-007](../../specs/spec-007-observability.md)). |
| REQ-301-13 | SHOULD | Skip the lock entirely when the token is far from expiry, acquiring it only when a refresh may be needed — with the check repeated inside the lock before refreshing. |
| REQ-301-14 | MUST | Behave correctly for a user whose record is absent: fail with a not-consented error, without creating a record. |

## Interface sketch

```js
// src/guard.js
/**
 * The only way an adapter obtains a token.
 *
 * @param {string}   userId
 * @param {string}   operationId       for scope coverage (spec-004)
 * @param {(token: Secret) => Promise<T>} fn
 * @returns {Promise<T>}
 *
 * @throws {ScopeInsufficientError} coverage check failed (REQ-301-05)
 * @throws {NotConsentedError}      no record for this user (REQ-301-14)
 * @throws {TokenDeadError}         invalid_grant; re-consent required (REQ-301-08)
 * @throws {TransientRefreshError}  refresh or lock wait failed transiently
 */
async function withValidToken(userId, operationId, fn) {}
```

Shape of the implementation, for the tests to target:

```text
withValidToken(user, op, fn):
  assertCoverage(op, record.grantedScopes)        # before any refresh (REQ-301-05)
  if not needsRefresh(record):                    # fast path (REQ-301-13)
      return fn(token)
  lock = acquireUserLock(user, ttl)               # (REQ-301-02)
  if lock is null:
      await otherHolderResult(bounded)            # (REQ-301-03, REQ-301-04)
      return fn(freshToken)
  try:
      re-read record; if still needsRefresh: refresh()
      return fn(freshToken)
  finally:
      release(lock)                               # every path (REQ-301-11)
```

The re-read inside the lock is not redundant: between the fast-path check and acquiring the lock,
another holder may already have refreshed.

## Acceptance criteria

- [ ] An API call with an expired token succeeds without any sweep having run.
- [ ] With Redis stopped, an API call still refreshes and succeeds.
- [ ] Twenty concurrent calls for one user with an expired token produce exactly one exchange, and
      all twenty succeed.
- [ ] A call needing a scope the user has not granted fails before any refresh or network call.
- [ ] A call for an unknown user fails with a not-consented error and creates nothing.
- [ ] `invalid_grant` fails the call once, marks `DEAD`, and does not retry.
- [ ] No adapter can be constructed with a raw token — asserted by a test, not by convention.
- [ ] A valid-token call adds no token-endpoint traffic.

## Tests

| Test ID | Covers | What it proves |
|---|---|---|
| T-301-01 | REQ-301-01 | With the sweep disabled and an expired token, an API call refreshes and succeeds. |
| T-301-02 | REQ-301-02 | A refresh interleaved between another holder's check and refresh cannot produce two exchanges. |
| T-301-03 | REQ-301-03 | Twenty concurrent calls for one expired token produce one exchange; all callers get a valid token. |
| T-301-04 | REQ-301-04 | A holder that never finishes causes waiters to time out with a transient error, not to hang. |
| T-301-05 | REQ-301-05 | An operation lacking scope coverage throws before any refresh and before any HTTP request. |
| T-301-06 | REQ-301-06 | With Redis unavailable, the guard still refreshes and the call succeeds. |
| T-301-07 | REQ-301-07 | When refresh is impossible, the call fails; the expired token is never used. |
| T-301-08 | REQ-301-08 | `invalid_grant` marks `DEAD`, alerts, fails the call, and attempts exactly one exchange. |
| T-301-09 | REQ-301-09 | No adapter export accepts a token directly; a constructed adapter cannot call Google without the guard. |
| T-301-10 | REQ-301-10 | A far-from-expiry call issues zero token-endpoint requests and acquires no lock. |
| T-301-11 | REQ-301-11 | The lock is released after success, after a thrown error, and after a timeout. |
| T-301-12 | REQ-301-12 | The guard's decision log and the API call log share a correlation id. |
| T-301-13 | REQ-301-13 | The fast path skips the lock; the slow path re-checks inside the lock and skips a redundant refresh when another holder already refreshed. |
| T-301-14 | REQ-301-14 | An unknown user throws `NotConsentedError` and no record is created. |

## Out of scope

- Lock implementation and TTL mechanics — [spec-300](spec-300-multi-user-queue-worker.md).
- The refresh exchange itself — [spec-002](../../specs/spec-002-refresh-engine.md).
- Which scopes an operation needs — [spec-004](../../specs/spec-004-scope-registry.md).
- Application logic that calls the adapters.

## References

- [`homelab/docs/architecture.md`](../docs/architecture.md)
- [ADR 0006](../../docs/adr/0006-bullmq-queue-for-multi-user.md)
