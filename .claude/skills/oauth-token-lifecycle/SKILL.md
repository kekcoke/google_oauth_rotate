---
name: oauth-token-lifecycle
description: Google OAuth refresh semantics as this project implements them — the state machine, skew windows, invalid_grant handling, single-flight refresh, and the 7-day Testing-mode expiry. Use when working on refresh logic, debugging token failures, or reasoning about expiry.
---

# OAuth token lifecycle

## States

`ABSENT` → `VALID` → `NEAR_EXPIRY` → `EXPIRED`, with `REFRESHING`, `SCOPE_INSUFFICIENT`,
`LOCKED_OUT` and `DEAD` as the interesting ones. Only `VALID` and `DEAD` are *stored*; the rest are
derived from `expires_at`, the clock, and the granted scope set. Storing a derived state is how it
goes stale.

Full transition table with required behaviour: `docs/token-lifecycle.md`.

## The refresh decision

```text
refresh when   expiresAt - now <= skew        (default skew 10 min, tick 5 min)
refresh when   expiresAt is absent, null, or unparseable
otherwise      do nothing
```

That is the whole of Option 2's advantage over Option 1. Get four things right:

1. **`expires_at` is an absolute UTC instant** derived from the token response at the moment it
   arrived — never an assumed hour, never a duration, never a local-clock guess.
2. **The clock is injected.** Every interesting case is a time case.
3. **The window is inclusive at the boundary** — `expiresAt - now === skew` refreshes.
4. **Missing expiry means refresh now**, not "no refresh needed". Getting this backwards produces a
   worker that never refreshes and looks healthy.

## Single-flight

At most one refresh in flight per user. Concurrent callers **await the in-flight result** — they do
not start a second exchange and do not fail. In `mac/` this is in-process; in `homelab/` it is a
Redis lock with a TTL, held across the check *and* the refresh so the decision and the action are
atomic per user.

The lock needs a TTL because a worker that dies holding it must not block that account forever, and
it needs extension so a slow-but-alive holder is not pre-empted mid-refresh.

## `invalid_grant` — the important one

The refresh token is gone. **Never retried, at any level, under any backoff — including a queue's
own retry mechanism.** Mark `DEAD`, alert, keep the row for audit, and wait for a human to
re-consent.

Causes:

| Cause | Tell |
|---|---|
| Testing-mode 7-day expiry | Token age ≈ 7 days, nothing else changed |
| User revoked access | Age < 7 days; grant absent from their Google permissions page |
| Password change | User confirms one |
| Client secret rotated | Rotation happened recently |
| Client scope configuration changed | Console change |
| Too many live refresh tokens for the client/user pair | Many recent re-consents during development |

In Testing mode this happens **weekly, forever**. Design for it as routine: day-6 age warning,
one-command re-consent, `docs/runbooks/re-consent.md`. The exits are a Workspace-internal app or
Production + verification (`docs/production-verification.md`).

## Refresh response gotchas

- **A refresh response may omit `refresh_token`.** Keep the stored one. Nulling it is a
  self-inflicted outage.
- **Granted scopes come from the response**, not the request. A user can decline one scope and the
  flow still succeeds.
- Store the response's expiry, not `now + 3600`.

## Two independent 7-day clocks

| Clock | Symptom when it lapses |
|---|---|
| Refresh token (Testing mode) | `invalid_grant` on every refresh for that user — loud |
| Gmail `watch()` | **Silence.** No pushes, no errors, nothing notices |

They are unrelated. A user can hold a healthy token and a dead watch. Conflating them costs a day
of debugging.

## Clocks and hosts that sleep

A Mac closes its lid; a VM is paused. Timers do not fire while paused and do not catch up — elapsed
time is simply gone. So:

- Never count ticks to infer elapsed time; read the clock.
- Detect a jump (elapsed far exceeding the interval) and re-evaluate immediately rather than waiting
  for the next tick.
- If the host slept longer than the refresh token's life, expect `invalid_grant` on resume.

## Error handling summary

`invalid_grant` → permanent, no retry. `429`/`5xx`/network → transient, exponential backoff with
full jitter, honour `Retry-After`, cap attempts. `401` on an API call → refresh once, retry once,
then escalate. Insufficient scope → fail locally before calling Google. Unmapped → treat as
non-retryable and log for triage. Full table: `specs/spec-009-error-taxonomy.md`.

## References

- `docs/token-lifecycle.md` — the state machine
- `specs/spec-002-refresh-engine.md` — the requirements
- `homelab/specs/spec-301-refresh-on-demand-guard.md` — refresh before the call
- `docs/runbooks/re-consent.md`
