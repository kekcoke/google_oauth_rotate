---
description: Report token state, expiry, refresh-token age and watch status for one option or user
argument-hint: --option simple|mac|homelab [--user <user_id>]
---

Report token health for: $ARGUMENTS

## If the implementation does not exist yet

`simple/` and `homelab/` are still specification-only. For those, say so plainly, name the spec
that defines the health report (`specs/spec-007-observability.md`, REQ-007-04), and stop. Do not
simulate output — a fabricated health report is worse than none.

`mac/` is under active implementation against `mac/IMPLEMENTATION.md`. Gather real data; if a
command below fails, report the failure as the finding rather than falling back to prose.

## Gather

Prefer the health endpoint; fall back to the store only if it is unavailable.

```sh
curl -s localhost:${HEALTH_PORT:-8080}/health | jq
```

For `mac/`, when the endpoint does not answer, work down this ladder — each rung distinguishes a
different fault:

```sh
docker compose -f mac/docker-compose.yml ps gmail-worker     # running? restarting? exited?
docker compose -f mac/docker-compose.yml logs --tail=50 gmail-worker
docker volume inspect mac_tokens                             # store present at all?
docker run --rm -v mac_tokens:/data alpine ls -l /data        # tokens.db present, mode 600?
```

Read the last tick decision line from the logs — it carries whether a refresh happened and the
seconds remaining until expiry, which is the whole health report in one line when the endpoint is
down. A container in a restart loop with no `tokens.db` almost always means `TOKEN_ENC_KEY` was
not injected: the worker is designed to fail to start rather than create an empty store, so check
that `scripts/up.sh` was used rather than a bare `docker compose up`.

Never read the store by decrypting a token to inspect it. State, expiry and scopes are all
readable without touching the refresh token.

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
| `ready` / `secrets` | Not-ready or an unavailable backend explains everything else. In `homelab/` that is a sealed Vault; in `mac/` it is a missing `TOKEN_ENC_KEY` or an unreachable store |

## Verdict

One of: healthy; **re-consent needed** → `/reconsent` and `docs/runbooks/re-consent.md`; **watch
renewal needed** → `/watch-renew-check` and `docs/runbooks/watch-renewal.md`; **infrastructure
fault** → name the component; **degraded** → say precisely what and what it costs.

Read the two 7-day clocks separately. A healthy token with a dead watch is a common state and it
looks fine from a token-only view.

Never print a token value, or a prefix of one, in the report.
