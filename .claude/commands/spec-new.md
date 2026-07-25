---
description: Draft a new numbered spec and its test-plan rows from a described component or behaviour
argument-hint: <what the component or behaviour is> [--option simple|mac|homelab]
---

Draft a new specification for: $ARGUMENTS

Follow the `spec-driven-development` skill and `specs/README.md`.

## First, decide what this actually is

- A new **component** → a new spec
- A new **behaviour** of an existing component → a new requirement in that spec, not a new spec
- A choice between viable approaches with lasting consequences → an **ADR** in `docs/adr/`, and
  possibly no spec change at all

If it is not a new spec, say so and do that instead. Creating a spec for what should be one
requirement is the most common error here.

## Then

1. Read `specs/README.md`, `specs/template-spec.md`, and any spec this overlaps.
2. Take the next free number in the right range: 001–099 shared, 100–199 `simple/`, 200–299
   `mac/`, 300–399 `homelab/`.
3. Set `applies_to` honestly — do not list an option that cannot implement it (push notifications
   in `simple/`, multi-user locking in `mac/`).
4. Write requirements as observable, falsifiable behaviour with RFC-2119 levels. One behaviour
   each. Include the MUST NOTs — most of this project's requirements are prohibitions.
5. Write the interface sketch as signatures and data shapes only. **No function bodies.**
6. Write the test table: every MUST gets at least one test id, each test id maps to exactly one
   requirement.
7. Add the spec to the index in `specs/README.md`.
8. Add the new test ids to the test plan of **every** option in `applies_to`, with given/when/then
   rows and a named fixture.

## Before presenting

Check your own draft against the traceability rules and say explicitly which ones you verified.
Then list: the file created, the test plans updated, and anything you could not decide without
input — do not guess at a requirement's level or scope.
