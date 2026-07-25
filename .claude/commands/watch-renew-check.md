---
description: Check Gmail watch() registrations for expiry and renew those due, then verify push actually works
argument-hint: --option homelab [--user <user_id>]
---

Check and renew Gmail `watch()` registrations: $ARGUMENTS

Follow `docs/runbooks/watch-renewal.md`. Applies to `homelab/` only — `simple/` and `mac/` have no
public HTTPS ingress and use the polling fallback.

## Why this exists separately

`watch()` expires about 7 days after registration, on a clock entirely independent of the refresh
token's. **Its failure mode is silence**, not an error: a healthy token with a dead watch produces a
system that looks fine and notices no mail. That is why this is a scheduled check rather than
something to handle when it breaks.

## If the implementation does not exist yet

Say so, point at `specs/spec-006-watch-pubsub.md`, and stop. Do not fabricate expiry figures.

## Steps

1. **Report current state** per user: stored `watch_expiration`, hours remaining, stored
   `history_id`, and whether push is expected for them at all.
2. **Check token health first.** `watch()` needs `gmail.readonly` or `gmail.modify`. A `DEAD` token
   means `/reconsent` first — renewing with a dead token just fails.
3. **Renew anything under 48 hours** by calling `users.watch` with the topic name. Re-registering an
   active watch is safe and simply extends it; there is no separate renew call.
4. **Persist the returned `expiration` and `historyId`.**
5. **Verify end to end.** A successful `watch()` response is not proof the receiver works. Confirm a
   real change produces a push that is processed.
6. **Check for a lapse.** If the watch had already expired, pushes for that window are gone — do not
   assume nothing happened. Sync from the stored `historyId`. If Google returns `404`/`410` because
   the cursor is too old, do the bounded recovery sync and **record the gap**.

## If push is still silent after renewal

Work down the chain; each link fails silently in a different way:

topic exists → Gmail has publish rights on it → subscription exists and is push-type → endpoint URL
correct and publicly reachable over HTTPS → receiver verifies the OIDC token → receiver returns 2xx
promptly → dead-letter topic configured.

**A growing unacked backlog on the subscription is the single most useful signal**: it means Gmail is
publishing and the receiver is the problem.

## Report

Per user: hours remaining before and after, whether renewal was performed, whether a lapse occurred
and was recovered, and the end-to-end verification result. Then flag any user for whom push is
expected and no watch exists — that is the silent failure this command exists to catch.
