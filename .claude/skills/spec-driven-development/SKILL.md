---
name: spec-driven-development
description: How specs, requirement IDs, test IDs and per-option test plans fit together in this repository. Use when writing or changing a spec, adding a requirement, reviewing spec quality, or checking traceability before implementation.
---

# Spec-driven development

Code in this repository exists because a numbered requirement asked for it, and is proven by a
test written before it. This skill is the mechanics.

## When to use

- Adding a component → new spec
- Adding a behaviour to an existing component → new requirement in that spec
- Choosing between viable approaches with lasting consequences → **ADR**, not a spec
- Before `/tdd-red`, to confirm the spec is complete enough to test against

## The identifier system

| Kind | Format | Range |
|---|---|---|
| Spec | `SPEC-NNN` | 001–099 shared, 100–199 `simple/`, 200–299 `mac/`, 300–399 `homelab/` |
| Requirement | `REQ-NNN-nn` | — |
| Test | `T-NNN-nn` | — |

Numbers are permanent. Retire with `Superseded by …` or `Withdrawn`; never renumber, never reuse.
Old commits, test names and review comments must keep resolving.

## The five traceability rules

1. Every MUST has ≥1 test id.
2. Every test id traces to exactly one requirement.
3. Every test id appears in the test plan of each option in `applies_to`.
4. Tests are written before implementation.
5. Changing a shared spec means updating every applicable test plan **in the same change**.

Rule 3 is the one that rots. `specs/spec-002` applies to `mac/` and `homelab/`, so a new
`REQ-002-20` needs its test id in both `mac/specs/test-plan.md` and
`homelab/specs/test-plan.md`. `/option-status` reports the drift.

## Writing a good requirement

Observable, falsifiable, one behaviour, no implementation choice, RFC-2119 level.

| Bad | Why | Better |
|---|---|---|
| MUST handle expiry correctly | Not falsifiable | MUST refresh when `expiresAt - now <= skew`, and not otherwise |
| MUST use Redis for locking | Implementation choice | MUST allow at most one in-flight refresh per user across the deployment (Redis is the ADR's answer) |
| MUST store and encrypt the token and log the outcome | Three behaviours | Three requirements |
| SHOULD be secure | Meaningless | Name the specific prohibition and test it |

Prohibitions are requirements too, and this project is full of them: no retry on `invalid_grant`,
no token in a log, no bypass of the guard, no plaintext refresh-token column. Phrase them as
MUST NOT and give them tests that prove absence.

## Spec sections

Purpose → Definitions (only load-bearing terms) → Requirements → Interface sketch (signatures and
data shapes, **never** a function body) → Acceptance criteria (human-runnable checks) → Tests
(id → requirement → what it proves) → Out of scope → References.

**Out of scope is not optional.** It is what stops an implementation pass wandering.

## Status lifecycle

`draft` → `accepted` → `implemented` → `superseded`. `implemented` means every MUST has a passing
test in at least one applicable option — not that someone wrote the code.

## Common failures

- A MUST with no test id. `/spec-review` fails it.
- A test id covering two requirements — the requirements are entangled; split them.
- A shared spec changed without touching the option test plans.
- Implementation detail leaking into a spec because it felt concrete.
- A requirement that would need `sleep` to test — it is phrased wrong; inject the clock instead.
- An `applies_to` list including an option that cannot possibly implement it (push notifications in
  `simple/`).

## References

- `specs/README.md` — the authoritative rules
- `specs/template-spec.md`, `specs/template-test-plan.md`
- `docs/adr/README.md` — when it is an ADR instead
