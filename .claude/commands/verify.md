---
description: Run the full quality gate — tests, lint, markdownlint, traceability and secret hygiene
argument-hint: [--option simple|mac|homelab]
---

Run the verification gate$ARGUMENTS.

Run every check, then report. Do not stop at the first failure — a partial report hides the other
problems.

## 1. Tests

Run from the option directory — the repository root has no package and no tests, so running there
finds nothing and reports the vacuous pass this command exists to prevent.

```sh
cd mac                                           # or the option under test
node --test                                      # unit, always
node --test --test-name-pattern integration      # needs Docker
npx c8 --check-coverage node --test              # once thresholds are configured
```

For `--option mac` this is a **hard gate**, not a courtesy check. `mac/` is under active
implementation against `mac/IMPLEMENTATION.md`:

- A failing or absent unit run is a fail, never a "not started".
- Docker Desktop is a prerequisite on this machine, so an unavailable-Docker skip on the
  integration run is itself a finding — say Docker is not running and fail the check.
- Coverage below the threshold committed in `mac/.c8rc.json` is a fail. Do not lower a threshold
  to pass.

For `simple/` and `homelab/`, which remain specification-only, say so plainly rather than
reporting a vacuous pass.

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

Note that `\.env\.` matches `.env.example`, which every option tracks on purpose — filter it with
`grep -v '\.env\.example$'` or the check fails on a clean tree.

Then, if images are built, confirm no credential in any layer.

## 4a. Deployment gates — `--option mac` only

Skip these when no image has been built yet; once one has, each is a fail, not a warning.

```sh
docker history --no-trunc gmail-worker | grep -i 'TOKEN_ENC_KEY' && echo FAIL
docker volume ls | grep -q mac_tokens || echo "FAIL: named volume missing"
grep -q 'restart: always' mac/docker-compose.yml || echo "FAIL: restart policy"
grep -q '127.0.0.1:8080:8080' mac/docker-compose.yml || echo "FAIL: health published off-loopback"
```

Also confirm the health endpoint refuses a connection from the LAN address, and that a second
worker against the same volume exits non-zero rather than starting.

## 5. Documentation consistency

- Relative links resolve across `docs/`, `specs/`, ADRs and runbooks
- No spec contradicts an accepted ADR
- Any doc claiming a behaviour the code does not have is a finding

## Output

A table of check → pass/fail → detail. Then a verdict: ready to commit, or the specific blocking
list. Report failures with their actual output; never summarise a failing suite as "mostly
passing".
