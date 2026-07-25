---
description: Run the full quality gate — tests, lint, markdownlint, traceability and secret hygiene
argument-hint: [--option simple|mac|homelab]
---

Run the verification gate$ARGUMENTS.

Run every check, then report. Do not stop at the first failure — a partial report hides the other
problems.

## 1. Tests

```sh
node --test                                     # unit, always
node --test --test-name-pattern integration      # needs Docker; skip with a note if unavailable
npx c8 --check-coverage node --test              # if coverage thresholds are configured
```

If there is no implementation yet, say so plainly rather than reporting a vacuous pass.

## 2. Lint

```sh
npx eslint .
npx markdownlint-cli '**/*.md' --ignore node_modules
```

## 3. Traceability

Run `/option-status` and include its result. Zero drift is required: every MUST has a test id,
every test id is in the applicable test plans, no orphans.

## 4. Secret hygiene

```sh
git ls-files | grep -E '\.env$|\.env\.|\.token$|credentials\.json|client_secret' || echo "clean"
git diff --cached | grep -nEi 'refresh_token|client_secret|BEGIN [A-Z ]*PRIVATE KEY' || echo "clean"
```

Then, if images are built, confirm no credential in any layer.

## 5. Documentation consistency

- Relative links resolve across `docs/`, `specs/`, ADRs and runbooks
- No spec contradicts an accepted ADR
- Any doc claiming a behaviour the code does not have is a finding

## Output

A table of check → pass/fail → detail. Then a verdict: ready to commit, or the specific blocking
list. Report failures with their actual output; never summarise a failing suite as "mostly
passing".
