# ADR 0005 — Treat 7-day refresh-token expiry as a normal condition

**Status:** Accepted

## Context

While an OAuth client's publishing status is **Testing**, Google expires issued refresh tokens
after 7 days. The project starts in Testing mode and the availability of a Google Workspace
tenant — the cheap route to long-lived tokens — is unconfirmed. Verification for Production
with restricted scopes requires review plus an annual CASA assessment, which is weeks of work.

There is no engineering trick that defeats this. A worker with perfect expiry-aware refresh
logic still stops working on day 8.

The failure is also not unique to Testing mode. Revocation, password changes, client-secret
rotation and exceeding the per-client refresh-token limit all produce the same
`invalid_grant`. Designing for the weekly case therefore hardens the system against all of them.

## Decision

`invalid_grant` is a **specified, tested, alerted state** (`DEAD` in
[`token-lifecycle.md`](../token-lifecycle.md)), not an exception handler. Concretely:

- The refresh engine distinguishes `invalid_grant` from transient errors and **stops retrying**
  it immediately.
- Monitoring tracks refresh-token **age** and warns at day 6, before day 7 breaks anything.
- `/reconsent` is a first-class maintenance command with a runbook, expected to be used weekly.
- A `DEAD` token's row is retained for audit; user data is not deleted.
- Documentation carries both exits — Workspace internal, or Production + verification.

## Consequences

### Good

- The weekly event is boring: alert, run one command, done.
- Retry logic cannot melt down against a permanently dead grant, because the taxonomy forbids
  retrying `invalid_grant`.
- The same machinery covers revocation and rotation, which would otherwise be untested paths.

### Bad

- The deployment is not truly unattended until an exit is taken. A human must re-consent
  roughly weekly, and if they are on holiday the automation is down.
- Extra scheduled jobs and alerting exist solely because of the publishing status, and they
  become dead weight after an exit — they must then be **deleted**, not left running.

## Alternatives considered

- **Pursue verification first, build second.** Blocks all development on a multi-week external
  process.
- **Ignore it and let the automation break weekly.** Produces silent data gaps rather than an
  alert, which is worse than the 7-day limit itself.
- **Persist a long-lived credential some other way** (service account, stored password).
  Service accounts cannot reach a consumer Gmail mailbox without Workspace domain-wide
  delegation; storing a password is not an option.

## Superseded by

Nothing yet. When the app becomes a Workspace internal app or passes verification, add the
superseding ADR, delete the day-6 warning and the weekly re-consent schedule, and prune the
tests that only existed for them. See
[`production-verification.md`](../production-verification.md).
