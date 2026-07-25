---
name: tdd-workflow
description: The red-green-refactor loop for this repository — node:test with CommonJS, injected clocks, faked Google endpoints, and the specific concurrency and time cases this domain requires. Use when writing tests or implementing against a test plan.
---

# TDD workflow

Test plan → failing test → minimum implementation → refactor → verify. Nothing skips a step,
including "obvious" changes, because the obvious changes here are the time and concurrency ones
that are wrong in ways that only a test reveals.

## The loop

| Step | Command | Done when |
|---|---|---|
| Red | `/tdd-red <spec-id>` | Tests exist for the plan's ids and fail for the intended reason |
| Green | `/tdd-green <spec-id>` | They pass, with the minimum code |
| Refactor | — | Structure improved, tests still green, no behaviour change |
| Verify | `/verify` | Full suite, lint, markdownlint, traceability all clean |

**Confirm the red failure message.** A test that fails on a typo or a missing import has not
tested anything. A test that passes the moment it is written is broken — report it, do not tweak
it until it fails.

## Harness

- `node:test`, CommonJS, `node --test`; coverage via c8
- Tests mirror source: `src/refresh-engine.js` → `tests/refresh-engine.test.js`
- Test names begin with the test id: `T-002-06 concurrent callers cause one exchange`
- Integration tests (real Postgres, Redis, dev Vault) are tagged so the always-on unit run stays
  fast

## Two absolute rules

**Never sleep.** Time comes from an injected clock the test advances. A `setTimeout` or a
`sleep` in a test is a defect — it is slow, flaky, and cannot express the case that actually
matters (a host sleeping for two hours between ticks).

**Never call Google.** Fake at the HTTP boundary, so real status codes, `Retry-After` headers and
error bodies drive the code. Do not stub the function under test; that tests the mock.

## Cases this domain requires

Time:

- Boundary equality: `expiresAt - now === skew` exactly, plus each side
- Absent, null and unparseable `expiresAt` — all mean refresh now
- Clock jumping hours between ticks (host sleep, paused VM)
- A non-UTC process timezone

Concurrency:

- N concurrent callers for one user → **exactly one** token exchange (assert the call count)
- Two different users → two exchanges, not serialised behind one lock
- A lock whose holder died → reclaimed at the TTL, and the reclaimer finishes the work
- Two overlapping sweeps → one exchange per due token

Failure:

- `invalid_grant` → exactly one attempt, ever (assert the count)
- `429` with `Retry-After` honoured over computed backoff
- Backoff jittered — repeated runs differ
- Sealed secret backend → readiness false, **one** alert not N, no cached fallback
- One user broken, others unaffected

Prohibitions — prove absence:

- No token material in captured log output (search for the fixture values, and their prefixes)
- No plaintext refresh token in the persisted representation
- No adapter path reaching Google without the guard
- No credential in the built image

Data:

- A refresh response omitting `refresh_token` leaves the stored one intact
- Partially granted scopes: consent succeeded, the set is narrower
- Duplicate and out-of-order Pub/Sub delivery → one net effect
- Multi-page list results → all pages, or an exposed page token

## Green means minimum

Write the least code that passes. Resist implementing the next requirement while you are in there;
it will not have a failing test, which makes it unproven code that looks tested. If you notice
something genuinely broken outside the current test, note it and raise it — do not fix it silently
in a green step.

## Fixtures

Named, shared, and never containing a real credential. Token values are obviously fake and exist
partly so redaction tests have something to search for. Every test-plan row names a fixture or
says `none`.

## References

- `mac/specs/test-plan.md` and `homelab/specs/test-plan.md` — the fixture catalogues
- `specs/template-test-plan.md`
- `specs/spec-009-error-taxonomy.md` — what each failure class must do
