# Runbook — Gmail `watch()` renewal

**When:** the daily `watch-renew-check` reports a subscription expiring within 48 hours, or
push notifications have stopped arriving.

**Applies to:** `homelab/` only. `simple/` and `mac/` have no public HTTPS ingress and use the
polling fallback in [`spec-006`](../../specs/spec-006-watch-pubsub.md).

**This is a second, independent 7-day clock.** Gmail's `watch()` registration expires about a
week after it is created, regardless of token health. A perfectly valid token with an expired
`watch()` produces silence, not an error — which is why this has its own scheduled check.

## Symptoms

- No Pub/Sub deliveries for longer than the mailbox's normal quiet period
- `watch-renew-check` reports `expiration` within the warning window, or missing entirely
- History-based sync falling behind: the stored `historyId` is far behind the mailbox's current

## Procedure

1. **Check what is actually registered.**

   ```text
   /watch-renew-check --option homelab --user <user_id>
   ```

   It reports the stored `historyId`, the last known `expiration`, and time remaining.
2. **Confirm the token is healthy first.** `watch()` needs `gmail.readonly` or `gmail.modify`.
   If the token is `DEAD`, do [`re-consent.md`](re-consent.md) first — renewing with a dead
   token just fails.
3. **Re-register.** Call `users.watch` with the Pub/Sub topic name. Re-registering an active
   watch is safe and simply extends it; there is no separate "renew" call.
4. **Store the new `expiration` and `historyId`** returned by the call. The `historyId` is the
   resume point for `users.history.list`.
5. **Verify end to end.** Send a message to the mailbox and confirm a push arrives at the
   receiver and is processed. A successful `watch()` response is not proof the receiver works.

## Recovering missed changes

If the watch lapsed, pushes for that window are gone. Do not assume nothing happened:

1. Take the stored `historyId` from before the lapse.
2. Call `users.history.list` from it and process the delta.
3. If Google returns `404`/`410` — the `historyId` is too old, typically beyond about a week of
   history — fall back to a bounded full sync (query by date range rather than the whole
   mailbox) and record that a gap occurred.

Handlers must be idempotent, because replaying history re-delivers changes that were already
processed. This is a requirement in [`spec-006`](../../specs/spec-006-watch-pubsub.md), not a
nicety.

## If push is still silent after renewal

Work down the chain; each link fails silently in a different way.

| Check | How | Failure looks like |
|---|---|---|
| Topic exists | Console / `gcloud pubsub topics describe` | `watch()` fails outright |
| Gmail can publish | `gmail-api-push@system.gserviceaccount.com` has `roles/pubsub.publisher` on the topic | `watch()` fails with a permission error |
| Subscription exists and is push-type | Console | `watch()` succeeds, nothing arrives |
| Push endpoint URL correct and publicly reachable over HTTPS | curl from outside the network | Subscription accumulates unacked messages |
| Receiver authenticates the OIDC token | Receiver logs | Deliveries rejected with `401`; subscription backlog grows |
| Receiver returns 2xx promptly | Receiver logs | Redelivery storms, duplicates |
| Dead-letter topic | Console | Failures disappear quietly |

A growing unacked backlog on the subscription is the single most useful signal: it means Gmail
is publishing and the receiver is the problem.

## Prevention

- Renew when under 48 hours remain, not on the last day — a failed renewal then has room for a
  retry.
- Alert on *absence of pushes* over a mailbox-appropriate interval, not only on renewal
  failures. Silence is the failure mode.
- Keep `historyId` persisted on every processed batch, so a lapse is recoverable rather than a
  full resync.
