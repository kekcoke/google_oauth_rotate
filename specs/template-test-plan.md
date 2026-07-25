# Test Plan — `<option>`

Every test ID from the specs this option implements, elaborated into a scenario concrete enough
to write as a failing test without further decisions.

## Specs in scope

| Spec | Applies here because | Test IDs |
|---|---|---|
| `spec-NNN` (link it) | … | T-NNN-01 … T-NNN-nn |

## Levels

| Level | Means | Runs where |
|---|---|---|
| unit | Pure logic, no I/O, injected clock | `node --test`, always |
| integration | Real store or real container, faked Google | `node --test`, tagged; needs Docker |
| e2e | Real Google endpoint, real credentials | Manual, documented, never in CI |

Google's token endpoint is **never** called from an automated test. Faked at the HTTP boundary
so the retry and error-taxonomy paths are exercised for real.

## Cases

| Test ID | Covers | Level | Given | When | Then | Fixture |
|---|---|---|---|---|---|---|
| T-NNN-01 | REQ-NNN-01 | unit | … | … | … | `…` |

Rules: one row per test ID; "Then" is a single observable assertion, not a list of hopes; every
row names a fixture or says `none`.

## Fixtures

| Name | Shape | Used by |
|---|---|---|
| `token-valid` | Expiry well beyond the skew window, full scope set | … |
| `token-near-expiry` | Expiry inside the skew window | … |
| `token-expired` | Expiry in the past, refresh token intact | … |
| `token-dead` | Refresh exchange returns `invalid_grant` | … |
| `token-partial-scopes` | Granted set narrower than requested | … |
| `google-429` | `429` with `Retry-After` | … |
| `google-5xx` | `503`, then success on retry | … |
| `clock-jump` | Injected clock advancing hours between ticks (host sleep) | … |

Time is always injected. A test that sleeps is a broken test.

## Coverage gate

- Every MUST in every spec in scope has ≥1 row here.
- No row lacks a spec reference.
- `/option-status` reports zero drift.

## Not tested here

Behaviour deliberately excluded, and why — for example Google's own rate-limit accuracy, or
another option's requirements.
