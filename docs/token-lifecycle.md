# Token Lifecycle

The state machine every option implements. Requirements in
[`spec-002-refresh-engine.md`](../specs/spec-002-refresh-engine.md) and
[`spec-009-error-taxonomy.md`](../specs/spec-009-error-taxonomy.md) trace back to these
states; test plans must cover every transition marked **must test**.

## States

| State | Definition |
|---|---|
| `ABSENT` | No token row for this user. Never consented, or the row was purged. |
| `VALID` | `expires_at > now + skew`, and `granted_scopes` covers what the caller needs. |
| `NEAR_EXPIRY` | `now + skew >= expires_at > now`. Still usable, refresh proactively. |
| `EXPIRED` | `expires_at <= now`. Access token unusable; refresh token may still be good. |
| `SCOPE_INSUFFICIENT` | Token is live but `granted_scopes` lacks a scope the caller needs. |
| `REFRESHING` | A refresh is in flight for this user. Concurrent callers wait, they do not start a second one. |
| `DEAD` | Refresh attempt returned `invalid_grant`. The refresh token is gone. Only human re-consent recovers. |
| `LOCKED_OUT` | Repeated `429`/`5xx` from Google. Token is probably fine; back off and retry. |

`skew` is the safety margin. Default **10 minutes** — the source material's
`expires_at < now + 10 * 60 * 1000`. `simple/` has no skew concept because it refreshes
unconditionally.

## Transitions

```text
                      consent granted
        ABSENT ─────────────────────────────► VALID
          ▲                                   │  │
          │                                   │  │ clock advances
          │ store purged                      │  ▼
          │                                   │ NEAR_EXPIRY ──┐
          │                                   │      │        │
          │                          new feature     │ refresh│
          │                          needs scope     ▼        │
          │                                   │  REFRESHING ◄─┘
          │                                   │   │   │   │
          │                                   ▼   │   │   └── 429/5xx ──► LOCKED_OUT
          │                    SCOPE_INSUFFICIENT │   │                       │
          │                             │         │   └── invalid_grant ──► DEAD
          │                             │         │                          │
          │        incremental consent  │         │ success                  │
          │                             └────────►┴──────────► VALID         │
          │                                                                  │
          └──────────────────── re-consent (human) ◄─────────────────────────┘

        EXPIRED is NEAR_EXPIRY that was not caught in time: same handling,
        but callers must not use the access token while in it.
```

## Required behavior per transition

| From → To | Trigger | Required behavior | Must test |
|---|---|---|---|
| `ABSENT` → `VALID` | Consent completed with `access_type=offline` | Persist refresh token, access token, `expires_at`, and the **actual** granted scopes from the response — not the requested ones | yes |
| `VALID` → `NEAR_EXPIRY` | Clock advance | No action beyond the next tick noticing it | yes |
| `NEAR_EXPIRY` → `REFRESHING` | Sweep tick, or a call path checking before use | Acquire the per-user lock first; if already held, wait for the result instead of refreshing again | yes |
| `REFRESHING` → `VALID` | Google returns a new access token | Persist atomically. If the response includes a new refresh token, replace the stored one; if it does not, **keep the existing one** — do not null it | yes |
| `REFRESHING` → `DEAD` | `400`/`401` with `error: invalid_grant` | Mark `DEAD`, stop retrying, emit a re-consent alert, keep the row for audit. Do not delete the user's data | yes |
| `REFRESHING` → `LOCKED_OUT` | `429`, `500`, `503` | Exponential backoff with jitter, honour `Retry-After`. Cap attempts, then alert | yes |
| `LOCKED_OUT` → `REFRESHING` | Backoff elapsed | Retry at most N times, then escalate | yes |
| `VALID` → `SCOPE_INSUFFICIENT` | A caller requires a scope not in `granted_scopes` | Fail the call **before** hitting Google, with an actionable error naming the missing scope | yes |
| `SCOPE_INSUFFICIENT` → `VALID` | Incremental consent completed | Merge newly granted scopes into the stored set; the access token from that exchange replaces the old one | yes |
| `DEAD` → `VALID` | Human re-consent | Overwrite refresh token and scopes; clear the `DEAD` marker; record the re-consent timestamp | yes |
| any → `ABSENT` | User revoked access, or an operator purged the row | Revoke server-side where possible, then delete | no |

## Why `DEAD` is common, not exceptional

In **Testing** publishing status, Google expires refresh tokens **7 days** after issue. A
personal deployment therefore enters `DEAD` about once a week, forever, until the app becomes
a Workspace-internal app or passes verification. Other causes:

- The user revoked access in their Google Account settings
- The user changed their password (invalidates Gmail-scoped grants)
- The scope set for the client changed
- More than the allowed number of live refresh tokens were issued for the same
  client/user pair, evicting the oldest
- Client secret was rotated

Consequences for design:

1. `invalid_grant` handling and its alert path get tests, same as the happy path.
2. Re-consent is a documented, low-friction operation — see
   [`runbooks/re-consent.md`](runbooks/re-consent.md).
3. Monitoring watches *token age*, not just refresh failures, so day 6 produces a warning
   before day 7 produces an outage.

## Clock and skew rules

- Compare against `expires_at` stored as an absolute timestamp (UTC), derived from Google's
  `expiry_date`/`expires_in` at the moment of the response — never from a local assumption
  that a token lasts an hour.
- Treat the local clock as suspect: a host that sleeps (a Mac lid closing) resumes with a
  large jump. On resume, re-evaluate immediately rather than waiting for the next tick.
- Never refresh on a schedule *tighter* than the skew window; that is Option 1's flaw
  generalised.
- Add jitter to sweep timing in multi-worker deployments so replicas do not stampede.

## Observability minimums

Per [`spec-007-observability.md`](../specs/spec-007-observability.md), every transition emits
a structured event with `user_id`, `from_state`, `to_state`, `reason`, and
`seconds_until_expiry`. **No token material, ever** — not the refresh token, not the access
token, not a prefix of either.
