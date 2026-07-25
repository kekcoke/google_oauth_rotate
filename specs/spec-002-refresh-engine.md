---
id: SPEC-002
title: Refresh engine
status: draft
applies_to: [mac, homelab]
depends_on: [SPEC-001, SPEC-009]
---

# SPEC-002 — Refresh engine

## Purpose

Decides whether an access token needs refreshing, performs the exchange, and persists the
result. This is the component the whole project exists for, and the one whose failure modes are
least forgiving: refresh too eagerly and you waste quota; too late and requests fail mid-flight;
retry the wrong error and you hammer Google against a dead grant.

`simple/` is excluded — it refreshes unconditionally on a cron grid and owns no decision logic
([`spec-100`](../simple/specs/spec-100-cron-refresh-worker.md)).

## Definitions

- **skew** — the safety margin before expiry inside which a token is refreshed. Default
  **10 minutes**, configurable.
- **tick** — the interval at which the sweep evaluates tokens. Default **5 minutes**.
- **single-flight** — at most one refresh in flight per user; concurrent callers await the
  in-flight result rather than starting their own.
- **injected clock** — all time comes from a supplied `now()`, never `Date.now()` inline, so
  behaviour is testable.

## Requirements

| ID | Level | Requirement |
|---|---|---|
| REQ-002-01 | MUST | Refresh when `expiresAt - now <= skew`, and not otherwise. |
| REQ-002-02 | MUST | Treat an absent or unparseable `expiresAt` as "refresh now" rather than "no refresh needed". |
| REQ-002-03 | MUST | Take all current time from an injected clock. |
| REQ-002-04 | MUST | Guarantee single-flight per user: concurrent `getValidToken` calls for one user result in exactly one token exchange, and all callers receive the same result. |
| REQ-002-05 | MUST | Persist the new access token and `expiresAt` through [`spec-001`](spec-001-token-store.md) before returning it to a caller. |
| REQ-002-06 | MUST | Classify a failed exchange per [`spec-009`](spec-009-error-taxonomy.md) and act on the class: `invalid_grant` → mark `DEAD` and stop; transient → backoff and retry; other → fail the call without retry. |
| REQ-002-07 | MUST NOT | Retry an `invalid_grant` failure, at any level, under any backoff. |
| REQ-002-08 | MUST | Emit a re-consent alert on transition to `DEAD`, naming the affected `user_id` and the runbook. |
| REQ-002-09 | MUST | Re-evaluate all tokens immediately after a detected clock jump or process resume, rather than waiting for the next tick. |
| REQ-002-10 | MUST | Refuse to serve a token whose refresh failed, rather than returning a stale access token past its expiry. |
| REQ-002-11 | MUST | Never log token material, including on the error path. |
| REQ-002-12 | SHOULD | Apply jitter to tick timing so multiple replicas do not refresh simultaneously. |
| REQ-002-13 | SHOULD | Warn when `refreshTokenIssuedAt` is more than 6 days old while the client is in Testing status. |
| REQ-002-14 | SHOULD | Record refresh duration and outcome as metrics per [`spec-007`](spec-007-observability.md). |
| REQ-002-15 | MUST | Be idempotent under duplicate invocation: two sweeps overlapping in time MUST NOT produce two exchanges for one user (follows from REQ-002-04, tested separately at the sweep level). |

## Interface sketch

```js
// lib/refresh-engine.js

/**
 * @param {Object} deps
 * @param {() => Date}  deps.now          injected clock (REQ-002-03)
 * @param {number}      deps.skewMs       default 600000
 * @param {TokenStore}  deps.store
 * @param {OAuthClient} deps.oauth
 * @param {Locker}      deps.locker       single-flight primitive (REQ-002-04)
 * @param {Logger}      deps.logger
 */
function createRefreshEngine(deps) {}

/** True when `expiresAt - now <= skewMs`, or expiry is missing/unparseable. */
function needsRefresh(expiresAt, now, skewMs) {}

/**
 * Returns a usable access token, refreshing first if needed.
 * @throws {TokenDeadError}       refresh token is gone; re-consent required
 * @throws {ScopeInsufficientError} granted scopes do not cover `requiredScopes`
 * @throws {TransientRefreshError} retries exhausted; caller may retry later
 */
async function getValidToken(userId, requiredScopes) {}

/** One sweep pass over tokens expiring inside the window. */
async function sweep() {}
```

## Acceptance criteria

- [ ] `needsRefresh` is a pure function with exhaustive boundary tests, including equality.
- [ ] Ten concurrent `getValidToken` calls for one user produce exactly one token exchange.
- [ ] An `invalid_grant` response produces one alert, a `DEAD` record, and zero retries.
- [ ] A `429` with `Retry-After` produces a delayed retry that honours the header.
- [ ] Suspending the host for two hours and resuming triggers immediate re-evaluation.
- [ ] No test sleeps; every timing case uses the injected clock.
- [ ] With Google faked, a full day of simulated time performs one refresh per token lifetime.

## Tests

| Test ID | Covers | What it proves |
|---|---|---|
| T-002-01 | REQ-002-01 | `needsRefresh` is false when expiry is beyond the skew window. |
| T-002-02 | REQ-002-01 | `needsRefresh` is true inside the window, and at exactly `now + skew`. |
| T-002-03 | REQ-002-01 | An already-expired token needs refresh. |
| T-002-04 | REQ-002-02 | Null, empty and garbage `expiresAt` all yield "refresh now". |
| T-002-05 | REQ-002-03 | Advancing the injected clock alone changes the decision; no wall-clock dependence. |
| T-002-06 | REQ-002-04 | Ten concurrent calls for one user cause one exchange; all receive the same token. |
| T-002-07 | REQ-002-04 | Concurrent calls for two different users cause two exchanges, not serialised behind one lock. |
| T-002-08 | REQ-002-05 | The store is written before the token is returned; a caller that observes the returned token can read it back. |
| T-002-09 | REQ-002-06 | Each error class routes to its specified action, table-driven over the taxonomy. |
| T-002-10 | REQ-002-07 | `invalid_grant` triggers exactly one exchange attempt — asserted by call count. |
| T-002-11 | REQ-002-08 | The `DEAD` transition emits an alert containing `user_id` and the runbook reference, and no token material. |
| T-002-12 | REQ-002-09 | A clock jump past expiry between ticks triggers immediate re-evaluation. |
| T-002-13 | REQ-002-10 | After exhausted retries, `getValidToken` throws rather than returning the stale token. |
| T-002-14 | REQ-002-11 | No logger call during a failed refresh receives a value matching stored token material. |
| T-002-15 | REQ-002-12 | Two engine instances with jitter enabled do not tick in lockstep across simulated ticks. |
| T-002-16 | REQ-002-13 | A token issued 6 days ago produces an age warning; 5 days does not. |
| T-002-17 | REQ-002-14 | A successful and a failed refresh each emit the specified metric with the right outcome label. |
| T-002-18 | REQ-002-15 | Two overlapping sweeps produce one exchange per due token. |
| T-002-19 | REQ-002-05 | A refresh response omitting `refresh_token` updates the access token and expiry while retaining the stored refresh token (with [`spec-001`](spec-001-token-store.md)). |

## Out of scope

- Where the lock lives — in-process for `mac/`, Redis for `homelab/`
  ([`spec-300`](../homelab/specs/spec-300-multi-user-queue-worker.md)).
- Refresh-before-call as an architectural guard —
  [`spec-301`](../homelab/specs/spec-301-refresh-on-demand-guard.md).
- Obtaining the first refresh token — [`spec-003`](spec-003-consent-flow.md).
- Scope sufficiency logic — [`spec-004`](spec-004-scope-registry.md).

## References

- [ADR 0002](../docs/adr/0002-expiry-aware-refresh-over-fixed-cron.md)
- [`docs/token-lifecycle.md`](../docs/token-lifecycle.md)
