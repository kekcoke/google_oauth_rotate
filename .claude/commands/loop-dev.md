---
description: Self-paced spec-to-green development loop for one spec, iterating until its acceptance criteria pass
argument-hint: <spec-id> [--option simple|mac|homelab]
---

Drive the development loop for: $ARGUMENTS

Intended for use with the `loop` skill so a session iterates on its own cadence. Each pass advances
one spec toward `implemented` and stops when its acceptance criteria are met — or when it needs a
human.

## One iteration

1. **Assess.** `/option-status` for the target spec. Identify the highest-value untested MUST,
   preferring the correctness-critical specs over breadth elsewhere. For `mac/` those are
   `spec-002` (refresh engine), `spec-001` (token store), `spec-009` (error taxonomy) and
   `spec-200` (the worker itself) — **not** `spec-301`, which is homelab-only. For `homelab/`,
   `spec-301` (on-demand guard) replaces `spec-200` in that list.
2. **Spec check.** If the requirement is not testable as written, `/spec-review` it and fix the spec
   first. Do not implement against a vague requirement.
3. **Red.** `/tdd-red <spec-id> --tests <ids>` for a small batch — two to five test ids, not the
   whole spec. Confirm each fails for the intended reason.
4. **Green.** `/tdd-green <spec-id>`. Minimum code.
5. **Refactor.** Structure only, tests still green.
6. **Verify.** `/verify`. Everything must pass, not just the new tests.
7. **Commit.** Conventional message naming the requirement ids covered, e.g.
   `feat(mac): expiry-aware refresh decision (REQ-002-01..04)`.
8. **Report.** Requirement ids newly covered, what remains, and what the next iteration will take.

## Stop and ask a human when

- The spec is ambiguous in a way that changes behaviour, and guessing would produce plausible-wrong
  code.
- Two specs conflict, or the work would contradict an accepted ADR.
- A security rule from `docs/security-model.md` blocks the obvious implementation.
- The same test has failed three iterations running for different reasons — that is a design
  problem, not a coding problem.

Say what you need and stop. Do not loop on a blocked item.

## Pause for a human, then continue

These are **not** failures — they are steps `mac/IMPLEMENTATION.md` expects a person to take. Do
the work up to the gate, state precisely what is needed, and resume on the same spec afterwards
rather than abandoning the loop.

- **Google Cloud console work** — project, APIs, consent screen, test user, Desktop client, the
  quota table. Step 2 of `mac/IMPLEMENTATION.md`, and it blocks step 3's exit criteria.
- **A real browser consent.** Step 3 needs exactly one, and it is the only e2e in the option.
- **A re-consent** after the 7-day Testing-mode expiry. Routine here, not an incident — follow
  `/reconsent`.
- **A Vault unseal** (`homelab/` only).

An automated test still must never use a real credential or call Google. That rule is absolute and
is not what this section relaxes: it relaxes only the *manual, human-run* steps around the tests.

## Never

- Weaken or delete a test to make a pass. If a test looks wrong, say why and stop.
- Implement past the failing tests. Untested code that ships alongside tested code is the worst
  outcome of an automated loop.
- Mark a spec `implemented` while any MUST is untested.
- Report progress that does not exist. "Three requirements implemented, two failing" is useful; "good
  progress" is not.

## Loop exit

Stop when every MUST in the target spec has a passing test and its acceptance criteria are
demonstrably met. Then report the spec as ready for review and end the loop rather than drifting
into the next spec unasked.

For `mac/`, a spec-level exit is not the same as the deployment-level one. The option is not done
because the suite is green; it is done when the container is running, ticking, and observed
performing a real refresh — the step 3 milestone in `mac/IMPLEMENTATION.md`. If the loop reaches
spec completion without that having ever been demonstrated, say so in the exit report rather than
implying the option works.
