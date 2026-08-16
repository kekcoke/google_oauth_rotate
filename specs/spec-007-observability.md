---
id: SPEC-007
title: Observability
status: accepted
applies_to: [simple, mac, homelab]
depends_on: [SPEC-001]
---

# SPEC-007 — Observability

> **Partial in `simple/`.** That option is a shell script and a crontab with no token logic, so only
> REQ-007-01 (no token material in logs) and REQ-007-11 (diagnosable at the default level) apply
> there. The excluded test IDs are named in
> [`simple/specs/test-plan.md`](../simple/specs/test-plan.md) under "Not tested here".

## Purpose

A token worker that fails silently is worse than no worker: the automation stops, nothing
complains, and the gap is discovered later from missing data. This spec defines what the system
emits so that both *failure* and *absence of success* are visible — and, just as importantly,
what it must never emit.

The hardest constraint here is negative. Everything interesting to log is next to a credential.

## Definitions

- **Structured event** — a single JSON object per line, with stable field names.
- **Token material** — refresh token, access token, authorization code, PKCE verifier, client
  secret, Vault token.
- **Redaction marker** — the fixed string emitted in place of a secret, e.g. `[redacted]`.
- **Liveness vs readiness** — liveness means the process is running; readiness means it can
  actually serve a valid token.

## Requirements

| ID | Level | Requirement |
|---|---|---|
| REQ-007-01 | MUST NOT | Emit token material in any log, metric label, span attribute, error message, stack trace, or crash dump — including truncated or prefixed forms. |
| REQ-007-02 | MUST | Emit a structured event for every token state transition, with `user_id`, `from_state`, `to_state`, `reason`, `seconds_until_expiry`. |
| REQ-007-03 | MUST | Emit a structured event for every refresh attempt, with outcome (`success`, `invalid_grant`, `transient`, `other`), duration, and attempt number. |
| REQ-007-04 | MUST | Expose a health endpoint or equivalent check reporting, per user: token state, seconds until expiry, refresh-token age, last successful refresh. |
| REQ-007-05 | MUST | Distinguish liveness from readiness: a worker that cannot reach its store or secret backend MUST report not-ready even though the process is alive. |
| REQ-007-06 | MUST | Alert on `invalid_grant`, naming the user and linking [`re-consent.md`](../docs/runbooks/re-consent.md). |
| REQ-007-07 | MUST | Alert on **absence** of a successful refresh within a configured window, not only on explicit failures. |
| REQ-007-08 | MUST | Alert when refresh-token age exceeds 6 days and `OAUTH_PUBLISHING_STATUS` is `testing`; MUST NOT alert on age when it is `production` or `internal`. |
| REQ-007-09 | MUST | Alert when the secret backend is unavailable (a sealed Vault fails every user at once, and must be identifiable as one infrastructure event, not N token events). |
| REQ-007-10 | MUST | Include a correlation identifier on all events belonging to one refresh or one API operation. |
| REQ-007-11 | MUST | Default to a log level that does not require raising verbosity to diagnose a refresh failure — the useful fields are present at info. |
| REQ-007-12 | SHOULD | Record metrics: refresh count by outcome, refresh duration, seconds-until-expiry per user, queue depth and job failures (`homelab/`), watch expiry remaining. |
| REQ-007-13 | SHOULD | Rotate or bound log files so a long-running worker cannot fill the disk. |
| REQ-007-14 | MUST | Provide a logger wrapper that redacts values registered as secret, so redaction does not depend on every call site remembering. |

## Interface sketch

```js
// lib/logger.js
/** Wraps a base logger; values registered via lib/secret.js render as `[redacted]`. */
function createLogger({ level, stream, correlationId }) {}

// lib/health.js
/**
 * @returns {{
 *   live: boolean,
 *   ready: boolean,
 *   store: 'ok'|'unavailable',
 *   secrets: 'ok'|'unavailable'|'sealed',
 *   users: Array<{
 *     userId: string, state: string, secondsUntilExpiry: number,
 *     refreshTokenAgeDays: number, lastSuccessfulRefresh: string|null,
 *     watchExpiresInHours: number|null
 *   }>
 * }}
 */
async function healthReport() {}
```

`healthReport()` is the data behind the `/token-health` command and the daily maintenance check.

## Acceptance criteria

- [ ] A full test run's captured log output, grepped for the fixtures' token values, yields
      nothing.
- [ ] A deliberately provoked refresh failure is diagnosable from default-level logs alone.
- [ ] A sealed secret backend reports not-ready and raises one infrastructure alert.
- [ ] Stopping all refreshes without any error raises the absence alarm within the window.
- [ ] `/token-health` output is sufficient to decide whether re-consent is needed.
- [ ] One refresh's events share a correlation id.

## Tests

| Test ID | Covers | What it proves |
|---|---|---|
| T-007-01 | REQ-007-01 | Captured output of a full lifecycle exercise contains no fixture token value, nor any 8-character prefix of one. |
| T-007-02 | REQ-007-01 | An error thrown with a token-bearing object attached serialises with the redaction marker. |
| T-007-03 | REQ-007-02 | Each state transition emits its event with all required fields. |
| T-007-04 | REQ-007-03 | Success, `invalid_grant`, transient and other outcomes each emit the correct outcome label and duration. |
| T-007-05 | REQ-007-04 | The health report includes every required per-user field. |
| T-007-06 | REQ-007-05 | With the store unreachable, liveness is true and readiness false. |
| T-007-07 | REQ-007-06 | An `invalid_grant` alert names the user and links the runbook. |
| T-007-08 | REQ-007-07 | Simulated time passing the window with no successful refresh raises the absence alarm. |
| T-007-09 | REQ-007-08 | With `OAUTH_PUBLISHING_STATUS=testing`, age of 6 days alerts and 5 days does not; with `production` or `internal`, neither does. |
| T-007-10 | REQ-007-09 | A sealed secret backend raises one infrastructure alert rather than one alert per user. |
| T-007-11 | REQ-007-10 | All events from one refresh share a correlation id; two concurrent refreshes do not. |
| T-007-12 | REQ-007-11 | A refresh failure's cause is present in default-level output. |
| T-007-13 | REQ-007-12 | Each specified metric is emitted with the expected name and labels. |
| T-007-14 | REQ-007-14 | A call site that passes a registered secret directly still produces redacted output. |

## Out of scope

- Choice of log aggregation, metrics or alerting infrastructure; this spec defines the signals,
  not the vendor.
- Dashboards.
- Audit logging inside Vault — [`spec-008`](spec-008-secret-management.md).

## References

- [`docs/token-lifecycle.md`](../docs/token-lifecycle.md)
- [`docs/security-model.md`](../docs/security-model.md)
