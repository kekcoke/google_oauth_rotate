---
description: Report token state, expiry, refresh-token age and watch status for one option or user
argument-hint: --option simple|mac|homelab [--user <user_id>]
---

Report token health for: $ARGUMENTS

## If the implementation does not exist yet

Say so plainly, name the spec that defines the health report
(`specs/spec-007-observability.md`, REQ-007-04), and stop. Do not simulate output — a fabricated
health report is worse than none.

## Gather

Prefer the health endpoint; fall back to the store only if it is unavailable.

```sh
curl -s localhost:${HEALTH_PORT:-8080}/health | jq
```

For `homelab/`, also check the infrastructure, because a single infrastructure fault presents as
every user failing at once:

```sh
docker compose ps
docker compose exec vault vault status | grep -i sealed
```

`simple/` has no health report of its own — read its log file and failure counter, and say clearly
that per-user token state is not observable from this option by design.

## Report

| Field | Why it matters |
|---|---|
| `state` | `DEAD` means re-consent, now |
| `secondsUntilExpiry` | Negative or inside the skew window with no refresh in progress is a fault |
| `refreshTokenAgeDays` | **≥6 days in Testing mode is a warning; ≥7 is an outage waiting** |
| `lastSuccessfulRefresh` | Absence over more than a token lifetime is a fault even with no errors logged |
| `grantedScopes` | Narrower than expected means a scope was declined at consent |
| `watchExpiresInHours` | Under 48 → renewal due; null while push is expected → the watch is gone |
| `ready` / `secrets` | Not-ready or a sealed backend explains everything else |

## Verdict

One of: healthy; **re-consent needed** → `/reconsent` and `docs/runbooks/re-consent.md`; **watch
renewal needed** → `/watch-renew-check` and `docs/runbooks/watch-renewal.md`; **infrastructure
fault** → name the component; **degraded** → say precisely what and what it costs.

Read the two 7-day clocks separately. A healthy token with a dead watch is a common state and it
looks fine from a token-only view.

Never print a token value, or a prefix of one, in the report.
