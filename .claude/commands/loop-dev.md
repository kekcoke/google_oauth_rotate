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
   preferring the correctness-critical specs (`spec-002` refresh engine, `spec-301` on-demand guard,
   `spec-009` error taxonomy) over breadth elsewhere.
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
- A test would need a real credential or a real Google call.
- The work needs Google Cloud console changes, a Vault unseal, or a re-consent.
- Two specs conflict, or the work would contradict an accepted ADR.
- A security rule from `docs/security-model.md` blocks the obvious implementation.
- The same test has failed three iterations running for different reasons — that is a design
  problem, not a coding problem.

Say what you need and stop. Do not loop on a blocked item.

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
