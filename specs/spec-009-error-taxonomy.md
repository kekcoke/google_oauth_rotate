---
id: SPEC-009
title: Error taxonomy and retry policy
status: accepted
applies_to: [simple, mac, homelab]
depends_on: []
---

# SPEC-009 — Error taxonomy and retry policy

> **Partial by option.** REQ-009-08 (class preserved through queue retries) applies to `homelab/`
> only — the others have no queue. In `simple/` the taxonomy governs how the shell script reads HTTP
> status codes; the requirements that presuppose a token exchange or typed errors apply to the
> application it pokes. Each option's plan names the test IDs it excludes.

## Purpose

One place that decides what every Google failure *means* and what to do about it. Without this,
retry logic is scattered and inconsistent, and the two catastrophic mistakes become easy:
retrying a permanently dead grant forever, and giving up on a transient error that one delayed
retry would have fixed.

Every other spec that touches Google routes its failures through this taxonomy.

## Definitions

- **Class** — the category a failure is mapped to. The class, not the HTTP status, decides
  behaviour.
- **Permanent** — retrying cannot succeed. Requires human or configuration action.
- **Transient** — retrying may succeed after a delay.
- **Fatal-to-caller** — the operation fails now; the caller decides whether to retry later.

## Classes

| Class | Trigger | Behaviour |
|---|---|---|
| `DEAD_GRANT` | `invalid_grant` from the token endpoint | Permanent. Mark `DEAD`, alert, **zero retries**. |
| `SCOPE_INSUFFICIENT` | Local coverage check fails, or `403 insufficient_permission` | Permanent until incremental consent. No retry; name the missing scopes. |
| `RATE_LIMITED` | `429`, or `403` with a rate-limit reason | Transient. Honour `Retry-After`; otherwise exponential backoff with jitter. |
| `SERVER_ERROR` | `500`, `502`, `503`, `504` | Transient. Exponential backoff with jitter. |
| `NETWORK` | DNS failure, connection reset, timeout | Transient. Exponential backoff with jitter. |
| `UNAUTHENTICATED` | `401` on an API call with a token believed valid | Refresh once, then retry the call once. If it recurs, escalate — do not loop. |
| `BAD_REQUEST` | `400` other than `invalid_grant`, `404` on a known-good resource | Permanent. Fail the caller; this is a bug or a stale reference. |
| `BACKEND_UNAVAILABLE` | Sealed Vault, unreachable store | Transient at the infrastructure level. Fail closed, alert once for the backend, not once per user. |
| `UNKNOWN` | Anything unmapped | Treated as `BAD_REQUEST` (no retry) and logged for triage — an unmapped error MUST NOT silently become a retry loop. |

## Requirements

| ID | Level | Requirement |
|---|---|---|
| REQ-009-01 | MUST | Map every Google failure to exactly one class before any handling decision. |
| REQ-009-02 | MUST NOT | Retry `DEAD_GRANT`, at any level, under any backoff, including via a queue's own retry mechanism. |
| REQ-009-03 | MUST | Honour `Retry-After` when present, in preference to computed backoff. |
| REQ-009-04 | MUST | Use exponential backoff with full jitter for transient classes, with a documented base, multiplier and cap. |
| REQ-009-05 | MUST | Cap total attempts per operation, and escalate — not silently drop — on exhaustion. |
| REQ-009-06 | MUST | Refresh at most once in response to `UNAUTHENTICATED`, then retry the call at most once. |
| REQ-009-07 | MUST | Treat an unmapped error as non-retryable and log it for triage. |
| REQ-009-08 | MUST | Preserve the class through queue retries, so `homelab/` job retries obey the same policy as in-process retries. |
| REQ-009-09 | MUST | Emit one alert per backend-unavailable event, not one per affected user. |
| REQ-009-10 | MUST NOT | Include token material in any error message, property, or serialised form. |
| REQ-009-11 | MUST | Expose typed errors (`TokenDeadError`, `ScopeInsufficientError`, `TransientRefreshError`, `BackendUnavailableError`) so callers can branch without string matching. |
| REQ-009-12 | SHOULD | Apply a circuit breaker per user after repeated transient failures, so one broken account cannot consume the retry budget of the whole deployment. |
| REQ-009-13 | SHOULD | Record class and attempt count as metric labels ([`spec-007`](spec-007-observability.md)). |

## Interface sketch

```js
// lib/errors.js
const CLASS = {
  DEAD_GRANT: 'DEAD_GRANT',
  SCOPE_INSUFFICIENT: 'SCOPE_INSUFFICIENT',
  RATE_LIMITED: 'RATE_LIMITED',
  SERVER_ERROR: 'SERVER_ERROR',
  NETWORK: 'NETWORK',
  UNAUTHENTICATED: 'UNAUTHENTICATED',
  BAD_REQUEST: 'BAD_REQUEST',
  BACKEND_UNAVAILABLE: 'BACKEND_UNAVAILABLE',
  UNKNOWN: 'UNKNOWN',
};

/** @returns {{ class: string, retryable: boolean, retryAfterMs: number|null }} */
function classify(error) {}

/** Full jitter. @returns {number} milliseconds */
function backoffMs(attempt, { baseMs = 1000, factor = 2, capMs = 60000 }) {}

class TokenDeadError extends Error {}
class ScopeInsufficientError extends Error {}   // .missing: string[]
class TransientRefreshError extends Error {}    // .attempts: number
class BackendUnavailableError extends Error {}
```

Defaults: base 1s, factor 2, cap 60s, max 5 attempts for transient classes.

## Acceptance criteria

- [ ] A table-driven test enumerates every class with a representative faked failure.
- [ ] `invalid_grant` produces exactly one token-endpoint call, asserted by call count.
- [ ] `Retry-After: 30` results in a ~30s delay, not the computed backoff.
- [ ] Backoff delays are jittered — repeated runs do not produce identical sequences.
- [ ] A `401` produces at most one refresh and one retry.
- [ ] An unrecognised error does not retry.
- [ ] A queued job for a `DEAD_GRANT` failure is not retried by the queue either.

## Tests

| Test ID | Covers | What it proves |
|---|---|---|
| T-009-01 | REQ-009-01 | Every representative failure maps to its specified class (table-driven over all classes). |
| T-009-02 | REQ-009-02 | `invalid_grant` yields exactly one attempt; the retry helper refuses it. |
| T-009-03 | REQ-009-03 | `Retry-After` in seconds and as an HTTP date both produce the specified delay. |
| T-009-04 | REQ-009-04 | Delays grow exponentially, stay under the cap, and differ between runs (jitter). |
| T-009-05 | REQ-009-05 | Attempts stop at the cap and raise `TransientRefreshError` with the attempt count. |
| T-009-06 | REQ-009-06 | A persistent `401` causes one refresh and one retry, then escalates. |
| T-009-07 | REQ-009-07 | An unmapped error classifies as `UNKNOWN`, is not retried, and is logged for triage. |
| T-009-08 | REQ-009-08 | A queue job carrying a `DEAD_GRANT` failure is not re-enqueued. |
| T-009-09 | REQ-009-09 | Ten users failing on one sealed backend produce one backend alert. |
| T-009-10 | REQ-009-10 | No error's message, properties, or JSON form contains token material. |
| T-009-11 | REQ-009-11 | Each typed error is thrown for its class and is distinguishable by `instanceof`. |
| T-009-12 | REQ-009-12 | Repeated transient failures for one user open its circuit without affecting another user. |
| T-009-13 | REQ-009-13 | Class and attempt count appear as metric labels. |

## Out of scope

- Google's own quota values — [`docs/google-cloud-setup.md`](../docs/google-cloud-setup.md).
- Alert routing — [`spec-007`](spec-007-observability.md).
- BullMQ configuration specifics —
  [`homelab/specs/spec-300-multi-user-queue-worker.md`](../homelab/specs/spec-300-multi-user-queue-worker.md).

## References

- [`docs/token-lifecycle.md`](../docs/token-lifecycle.md)
- [ADR 0005](../docs/adr/0005-testing-mode-refresh-token-expiry.md)
