---
id: SPEC-006
title: Gmail watch and Pub/Sub push
status: draft
applies_to: [homelab]
depends_on: [SPEC-002, SPEC-004, SPEC-005]
---

# SPEC-006 — Gmail watch and Pub/Sub push

## Purpose

Real-time mailbox change notification, plus the polling fallback for deployments with no public
ingress. Two things make this harder than it looks: `watch()` expires after about 7 days — a
second clock entirely independent of the refresh token's — and its failure mode is **silence**,
not an error. A dead watch produces a working system that quietly stops noticing mail.

`homelab/` implements push. `mac/` implements the polling fallback (REQ-006-11 to REQ-006-13
only); `simple/` implements neither. Do not expose a laptop to the internet to make push work.

## Definitions

- **`watch()`** — `users.watch`, registering a Pub/Sub topic to receive mailbox change events.
- **`historyId`** — the mailbox change cursor; the resume point for `users.history.list`.
- **Push message** — a Pub/Sub delivery containing an email address and a `historyId`. It is a
  *hint that something changed*, never the change itself.
- **Lapse** — a period during which no watch was registered, so no pushes were sent.

## Requirements

| ID | Level | Requirement |
|---|---|---|
| REQ-006-01 | MUST | Persist the `expiration` and `historyId` returned by `watch()`. |
| REQ-006-02 | MUST | Re-register `watch()` when fewer than 48 hours remain before `expiration`. |
| REQ-006-03 | MUST | Alert when no watch is registered for a user that should have one, or when re-registration fails. |
| REQ-006-04 | MUST | Verify the push request's OIDC token and expected audience before processing; reject unauthenticated deliveries. |
| REQ-006-05 | MUST | Treat the push payload as untrusted: use it only as a trigger, and re-fetch state from Gmail via `users.history.list`. |
| REQ-006-06 | MUST | Process history from the **stored** `historyId`, not the pushed one, and advance the stored cursor only after successful processing. |
| REQ-006-07 | MUST | Be idempotent under duplicate and out-of-order delivery: Pub/Sub delivers at least once, and redelivery is normal. |
| REQ-006-08 | MUST | Acknowledge a delivery only after durably recording the work, or after deliberately deferring it to the queue; MUST NOT ack then lose the event. |
| REQ-006-09 | MUST | Handle a `404`/`410` from `users.history.list` (cursor too old) with a bounded recovery sync and a recorded gap, rather than an unbounded full mailbox read. |
| REQ-006-10 | MUST | Alert on absence of expected pushes over a configured interval — silence is the primary failure mode. |
| REQ-006-11 | MUST | Provide a polling mode that advances the same stored `historyId` using `users.history.list` on an interval, for deployments without ingress. |
| REQ-006-12 | MUST | Make push and polling mutually exclusive per user, and never process the same change through both. |
| REQ-006-13 | MUST | Reject at startup a polling interval below `POLL_INTERVAL_FLOOR_MS` (default 60000). The floor in use, and the Gmail quota figure it derives from, MUST be recorded in the quota table of [`docs/google-cloud-setup.md`](../docs/google-cloud-setup.md). |
| REQ-006-14 | MUST | Stop attempting `watch()` for a user whose token is `DEAD`, and resume after re-consent. |
| REQ-006-15 | SHOULD | Configure a dead-letter topic so repeated processing failures are visible rather than silently retried forever. |

## Interface sketch

```js
// lib/watch.js
/** @returns {Promise<{historyId: string, expiration: Date}>} */
async function registerWatch(userId, topicName) {}

/** @returns {Promise<{dueForRenewal: boolean, expiration: Date|null, remainingMs: number}>} */
async function checkWatch(userId) {}

// lib/push-receiver.js
/**
 * @throws {UnauthenticatedPushError} bad or missing OIDC token (REQ-006-04)
 * Ack semantics: resolve => ack, reject => nack (REQ-006-08)
 */
async function handlePush(request) {}

// lib/history-sync.js
/**
 * Advances the stored cursor only on success (REQ-006-06).
 * @throws {CursorTooOldError} history unavailable from the stored id (REQ-006-09)
 * @returns {Promise<{processed: number, newHistoryId: string}>}
 */
async function syncFrom(userId) {}
```

## Acceptance criteria

- [ ] A registered watch is renewed before expiry by the scheduled check, verified against an
      injected clock rather than by waiting a week.
- [ ] An unauthenticated push is rejected.
- [ ] The same push delivered three times produces one net effect.
- [ ] A simulated lapse is recovered from the stored cursor, with the gap recorded.
- [ ] A `410` from history triggers bounded recovery, not a full mailbox read.
- [ ] Polling mode produces the same processed set as push mode for an identical change stream.
- [ ] Turning off pushes entirely raises the silence alarm within the configured interval.

## Tests

| Test ID | Covers | What it proves |
|---|---|---|
| T-006-01 | REQ-006-01 | `expiration` and `historyId` from a faked `watch()` response are persisted. |
| T-006-02 | REQ-006-02 | With 47 hours remaining the check reports due; with 49 it does not. |
| T-006-03 | REQ-006-03 | A failed re-registration and a missing watch each raise an alert naming the user. |
| T-006-04 | REQ-006-04 | Pushes with a missing, malformed, or wrong-audience OIDC token are rejected. |
| T-006-05 | REQ-006-05 | Processing ignores the payload's `historyId` for fetching and calls history with the stored cursor. |
| T-006-06 | REQ-006-06 | A failure mid-processing leaves the stored cursor unchanged. |
| T-006-07 | REQ-006-07 | Triple delivery of one push, and two pushes delivered out of order, each produce one net effect. |
| T-006-08 | REQ-006-08 | A processing failure results in a nack; a success acks after the durable write. |
| T-006-09 | REQ-006-09 | A faked `410` triggers the bounded recovery path and records a gap. |
| T-006-10 | REQ-006-10 | No deliveries for longer than the configured interval raises the silence alarm. |
| T-006-11 | REQ-006-11 | Polling mode advances the same cursor and processes the same changes as push mode. |
| T-006-12 | REQ-006-12 | Enabling both modes for one user is rejected by configuration validation. |
| T-006-13 | REQ-006-13 | A configured interval below the floor fails startup naming both values; the default floor applies when unset; a value at or above it starts. |
| T-006-14 | REQ-006-14 | No `watch()` attempt is made for a `DEAD` token; the attempt resumes after re-consent. |
| T-006-15 | REQ-006-15 | Repeated failures route to the dead-letter path rather than retrying indefinitely. |

## Out of scope

- Pub/Sub topic and subscription creation — [`docs/google-cloud-setup.md`](../docs/google-cloud-setup.md).
- What to do with a mailbox change; this spec delivers the event.
- TLS termination and ingress for the receiver — [`homelab/docs/deploy-homelab.md`](../homelab/docs/deploy-homelab.md).
- Drive change notifications; only Gmail push is in scope.

## References

- [`docs/runbooks/watch-renewal.md`](../docs/runbooks/watch-renewal.md)
- [`docs/architecture-overview.md`](../docs/architecture-overview.md)
