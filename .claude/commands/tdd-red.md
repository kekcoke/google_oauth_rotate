---
description: Write the failing tests for a spec's test-plan rows — red phase, tests only, no implementation
argument-hint: <spec-id> [--option simple|mac|homelab] [--tests T-002-01,T-002-06]
---

Write failing tests for: $ARGUMENTS

Follow the `tdd-workflow` skill.

## Rules

- **Tests only.** No implementation. If a test cannot fail for the right reason without a module,
  create the minimum stub: exported names that throw `new Error('not implemented')`, nothing else.
- **One test per test id**, with the id at the start of the test name so a grep finds it.
- **One observable assertion per test.**
- **Never sleep.** Advance the injected clock explicitly.
- **Never call Google.** Fake at the HTTP boundary — real status codes, real `Retry-After`
  headers, real error bodies. Do not stub the function under test.
- **No real credentials in fixtures**, not even expired ones.
- **Assert call counts** wherever the requirement is "exactly once": single-flight, no-retry on
  `invalid_grant`, one alert not N.
- **Prove absence** for the prohibitions: no token in captured logs, no plaintext column, no
  bypass of the guard, no credential in the image.
- `node:test`, CommonJS, tests mirroring source layout.

## Steps

1. Read the spec and the option's test plan. Take the rows for the requested test ids, or all of
   them if none is specified.
2. Render each row faithfully — the given/when/then is already decided; do not reinterpret it. If a
   row is too vague to render as one unambiguous test, stop and say which row and what decision is
   missing.
3. Create or reuse fixtures from the plan's fixture catalogue. Reuse before inventing.
4. Run the tests.
5. **Read every failure message.** A failure caused by a typo, a missing import, or a bad path has
   not tested anything — fix the test. A test that **passes** immediately is broken: report it
   rather than adjusting it until it fails.

## Output

The test ids written, the file paths, the command to run them, and the failure output showing each
fails for the intended reason. Then state what `/tdd-green` has to make true.
