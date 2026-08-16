# Specifications

Every line of code in this repository exists because a numbered requirement asked for it, and
is proven by a test that was written before it. This directory holds the shared,
option-agnostic specs; each option (`simple/`, `mac/`, `homelab/`) adds its own and carries a
test plan.

## Index

| Spec | Title | Applies to |
|---|---|---|
| [spec-001](spec-001-token-store.md) | Token store | mac, homelab (assumed of the app in `simple/`) |
| [spec-002](spec-002-refresh-engine.md) | Refresh engine | mac, homelab |
| [spec-003](spec-003-consent-flow.md) | Consent flow | mac, homelab |
| [spec-004](spec-004-scope-registry.md) | Scope registry and incremental authorization | mac, homelab |
| [spec-005](spec-005-api-adapters.md) | Google API adapters | mac, homelab |
| [spec-006](spec-006-watch-pubsub.md) | Gmail watch and Pub/Sub | homelab; mac (polling fallback only) |
| [spec-007](spec-007-observability.md) | Observability | all |
| [spec-008](spec-008-secret-management.md) | Secret management | all |
| [spec-009](spec-009-error-taxonomy.md) | Error taxonomy and retry | all |

Option-specific specs: [`simple/specs/`](../simple/specs/), [`mac/specs/`](../mac/specs/),
[`homelab/specs/`](../homelab/specs/).

## Identifier scheme

| Kind | Format | Example |
|---|---|---|
| Spec | `SPEC-NNN` | `SPEC-002` |
| Requirement | `REQ-NNN-nn` | `REQ-002-03` |
| Test | `T-NNN-nn` | `T-002-07` |

Numbers are **permanent**. Never renumber, never reuse. A requirement that no longer applies is
marked `Superseded by REQ-…` or `Withdrawn`, with its number retained so old commits, tests and
review comments still resolve.

Number ranges: `001`–`099` shared, `100`–`199` `simple/`, `200`–`299` `mac/`, `300`–`399`
`homelab/`.

## Traceability rules

1. **Every MUST requirement has at least one test ID.** A MUST with no test is an incomplete
   spec, and `/spec-review` fails it.
2. **Every test ID traces to exactly one requirement.** A test covering two requirements means
   the requirements are entangled — split them.
3. **Every test ID is accounted for in the test plan of each option the spec applies to** — either
   as a covered row, or named in that plan's **Not tested here** section with a reason. Where the
   required behaviour differs between options, the plan says how. Silence is drift; an explicit
   exclusion is a decision. A spec that applies only partially to an option says so under its front
   matter, and that option's plan names the excluded IDs.
4. **Tests are written before implementation.** `/tdd-red` turns test-plan rows into failing
   tests; `/tdd-green` makes them pass. Code that arrived before its failing test is a process
   violation — delete it and start from the test.
5. **Changing a shared spec means revisiting every applicable test plan** in the same change.
   `/option-status` reports drift.

## Requirement language

RFC 2119. **MUST** is a hard requirement whose violation is a bug. **SHOULD** is a strong
default that may be traded away with a recorded reason. **MAY** is genuinely optional and does
not need a test.

Write requirements as observable behaviour, not implementation. "MUST persist the absolute
expiry timestamp derived from the token response" is testable. "MUST handle expiry correctly"
is not.

## Status values

`draft` → under discussion, may change freely.
`accepted` → agreed; changes need an ADR or a superseding requirement.
`implemented` → every MUST has a passing test in at least one option.
`superseded` → replaced; the front matter names the replacement.

## Adding a spec

Use [`template-spec.md`](template-spec.md) and [`template-test-plan.md`](template-test-plan.md),
or run `/spec-new`. Take the next free number in the right range, add it to the index above, and
list its test IDs in the affected options' test plans in the same commit.
