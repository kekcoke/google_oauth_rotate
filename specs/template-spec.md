---
id: SPEC-NNN
title: <Short noun phrase>
status: draft
applies_to: [simple, mac, homelab]
depends_on: []
---

# SPEC-NNN — `<Title>`

## Purpose

Two or three sentences: what this component is responsible for, and what breaks without it.
Link the architecture doc or ADR that motivates it. No implementation detail here.

## Definitions

Only terms whose meaning is load-bearing and might be read two ways. Delete this section if
there are none.

## Requirements

| ID | Level | Requirement |
|---|---|---|
| REQ-NNN-01 | MUST | Observable behaviour, stated so a test can falsify it. |
| REQ-NNN-02 | MUST | … |
| REQ-NNN-03 | SHOULD | A strong default that may be traded away with a recorded reason. |

Rules: one behaviour per requirement; no "and" joining two independently testable claims; no
implementation choices (that is what an ADR is for).

## Interface sketch

Signatures and data shapes only — enough for a test to be written against, not an
implementation. No function bodies.

```js
// lib/<component>.js
/**
 * @param {...} ...
 * @returns {Promise<...>}
 * @throws {NamedError} when …
 */
```

## Acceptance criteria

What must be demonstrably true for this spec to move to `implemented`. Phrased as checks a
human can run, not as restated requirements.

- [ ] …

## Tests

| Test ID | Covers | What it proves |
|---|---|---|
| T-NNN-01 | REQ-NNN-01 | One sentence. |
| T-NNN-02 | REQ-NNN-02 | … |

Every MUST above appears in this table at least once. Each option listed in `applies_to`
elaborates these IDs into given/when/then rows in its own test plan.

## Out of scope

What a reader might reasonably expect here and will not find, with a pointer to where it lives
instead. This section prevents scope creep during implementation.

## References

Links to ADRs, architecture docs, runbooks, and external documentation.
