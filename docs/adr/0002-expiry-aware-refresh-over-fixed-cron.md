# ADR 0002 — Expiry-aware refresh is the default

**Status:** Accepted

## Context

Option 1 refreshes every 30 minutes whether or not the access token needs it. That is simple
and, for a single user, mostly harmless — but it has three real defects:

1. It calls Google's token endpoint ~48 times a day per user to do work that is needed once an
   hour at most.
2. A fixed grid can still be wrong. If the worker or app is down at the moment the token
   expires, the next scheduled attempt may come after a request has already failed.
3. It makes the refresh decision invisible. Nothing in the system knows *why* a refresh
   happened, so nothing can alert on it not happening.

The alternative from the source material reads the stored expiry and refreshes only inside a
safety window:

```text
if expires_at < now + 10 minutes: refresh() else: do nothing
```

## Decision

Expiry-aware refresh is the default for `mac/` and `homelab/`, with a default skew of
**10 minutes** and a **5-minute** evaluation tick. `simple/` keeps the unconditional
30-minute cron, deliberately, as the minimal-integration option.

In `homelab/`, the timer is a safety net rather than the mechanism: any call path checks
expiry immediately before use and refreshes on demand
([`spec-301`](../../homelab/specs/spec-301-refresh-on-demand-guard.md)).

## Consequences

### Good

- Refresh calls drop to roughly one per token lifetime per user.
- The refresh decision becomes a testable pure function of `(expires_at, now, skew)`, which
  is where the interesting test cases live: boundary equality, clock jumps, negative skew.
- Refreshing only when needed means an unexpected refresh is a signal worth alerting on.

### Bad

- Correctness now depends on `expires_at` being stored accurately. A wrong or missing expiry
  turns "refresh only when needed" into "never refresh". Storing the absolute timestamp
  derived from the token response is therefore a MUST, not a SHOULD.
- The system depends on the local clock. A sleeping Mac resumes with a jump, so
  wake-up re-evaluation is required rather than optional.

## Alternatives considered

- **Refresh on 401 only (reactive).** Zero wasted calls, but every token expiry costs one
  failed user-visible request, and concurrent failures stampede the token endpoint.
- **Tighter cron (every 5 minutes, unconditional).** Multiplies the waste rather than fixing
  the blindness.
- **Trust `expires_in` = 3600 without storing it.** Google is not obliged to return the same
  lifetime, and any restart loses the in-memory value.

## References

- [`docs/token-lifecycle.md`](../token-lifecycle.md)
- [`specs/spec-002-refresh-engine.md`](../../specs/spec-002-refresh-engine.md)
