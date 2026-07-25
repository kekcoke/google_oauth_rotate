---
description: Implement the minimum code to pass the failing tests for a spec — green phase
argument-hint: <spec-id> [--option simple|mac|homelab]
---

Make the failing tests pass for: $ARGUMENTS

Follow the `tdd-workflow` skill.

## Preconditions — check before writing code

1. Failing tests exist for the requirement ids you are about to implement. If not, run `/tdd-red`
   first. Implementation without a preceding failing test is a process violation, and the fix is to
   delete the code and start from the test.
2. Run the suite and confirm the failures are the ones you intend to fix.

## Rules

- **Minimum code to pass.** Do not implement the next requirement while you are in here — it will
  have no failing test, which makes it unproven code that looks tested.
- **Do not weaken a test to make it pass.** If a test looks wrong, stop and say why; changing the
  test to match the code inverts the whole method.
- **Do not add behaviour the spec does not require.** Extra behaviour is untested behaviour.
- Node ≥18, **CommonJS**, lowercase-hyphen filenames, dependencies injected (clock, store, HTTP,
  logger) — never read directly.
- Never log token material, including on the error path.
- If you notice a real defect outside the current tests, note it and raise it; do not fix it
  silently in a green step.

## After green

1. Full suite passes, not just the new tests.
2. Refactor for structure with the tests still green — no behaviour change.
3. `npx eslint .` and `npx markdownlint-cli '**/*.md' --ignore node_modules` clean.
4. If anything about the spec turned out to be wrong or under-specified, say so — the spec is the
   artefact that outlives the code, and a spec that disagrees with working code is a defect.

## Output

Requirement ids now covered by passing tests, files created or changed, the suite result, and
anything you deliberately did not implement and why. Then run `/verify`.
