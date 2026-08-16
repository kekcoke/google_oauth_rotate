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

### `mac/` targets

`mac/` is a self-contained package under active implementation — see `mac/IMPLEMENTATION.md` for
the standing decisions, which are binding.

- Modules whose interface sketch lives in a **shared** spec go in `mac/lib/`; option-specific
  process code goes in `mac/src/` (`worker.js`, `instance-guard.js`, `index.js`,
  `consent-cli.js`). Tests mirror both: `mac/tests/lib/`, `mac/tests/src/`.
- Store is SQLite on a named volume. Refresh tokens are envelope-encrypted (AES-256-GCM) with the
  key id in the stored form, so rotation is possible.
- `TOKEN_ENC_KEY` arrives as an environment variable injected by `scripts/up.sh` from the macOS
  Keychain. The container cannot read the Keychain itself. Read it **only** through
  `lib/config.js` — `process.env` outside that module is a lint failure.
- Time comes from the injected clock. `Date.now()` in decision code is a lint failure, not a
  style preference.
- Single-flight is in-process. There is no Redis lock, no queue, no Vault, no ingress.
- If you notice a real defect outside the current tests, note it and raise it; do not fix it
  silently in a green step.

## After green

1. Full suite passes, not just the new tests.
2. Refactor for structure with the tests still green — no behaviour change.
3. `npx eslint .` and `npx markdownlint-cli '**/*.md' --ignore node_modules` clean. Run eslint
   from the option directory; it needs the `package.json` and flat config that step 2 of
   `mac/IMPLEMENTATION.md` creates. If they do not exist yet, that is the blocker to report —
   do not skip the check.
4. If anything about the spec turned out to be wrong or under-specified, say so — the spec is the
   artefact that outlives the code, and a spec that disagrees with working code is a defect.

## Output

Requirement ids now covered by passing tests, files created or changed, the suite result, and
anything you deliberately did not implement and why. Then run `/verify`.
