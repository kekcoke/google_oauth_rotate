---
description: Report requirement to test to implementation coverage per option, and any traceability drift
argument-hint: [--option simple|mac|homelab]
---

Report coverage and drift$ARGUMENTS.

Read the specs, the test plans, and the tests that exist. Produce the report — do not fix anything;
this command is a mirror.

## Gather

1. All specs: `specs/spec-*.md` and `*/specs/spec-*.md`. For each, its requirement ids with levels
   and its `applies_to` list.
2. All test plans: `*/specs/test-plan.md`. For each, the test ids present.
3. Tests on disk, matched by the test id in each test name.

## Report

### Per option

| Spec | MUSTs | Test ids planned | Tests written | Passing | Status |
|---|---|---|---|---|---|

### Drift — each of these is a defect

- **Untested MUSTs**: requirement ids with no test id anywhere.
- **Missing rows**: test ids in a spec but absent from a test plan of an option in its
  `applies_to`.
- **Orphan tests**: test ids in a test plan or in code with no matching requirement.
- **Multi-mapped tests**: one test id covering two requirements.
- **Unimplemented plans**: test ids planned with no test written.
- **Failing tests**, listed by test id.
- **Status lies**: a spec marked `implemented` with an untested MUST, or `draft` with a full
  passing suite.
- **Renumbering**: requirement or test ids that changed identity relative to `main`.

### Next actions

The two or three highest-value moves, in order. Prefer untested MUSTs on the correctness-critical
specs — `spec-002` (refresh engine), `spec-301` (on-demand guard), `spec-009` (error taxonomy) —
over breadth elsewhere.

## Honesty rules

Count what exists, not what is intended. A test plan row is not a test. A written test is not a
passing test. If a spec has no implementation at all, say `not started` — never imply progress
that is not there.
