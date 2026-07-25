---
description: Review a spec for traceability, testability and domain correctness before implementation starts
argument-hint: <spec-id or path> (omit to review everything changed on this branch)
---

Review: $ARGUMENTS

If no argument is given, review every spec and test plan changed on this branch against `main`.

## Mechanical checks — report each as pass or fail

1. Frontmatter present and valid: `id`, `title`, `status`, `applies_to`, `depends_on`.
2. Every MUST and MUST NOT has ≥1 test id.
3. Every test id maps to exactly one requirement.
4. Every test id appears in the test plan of every option in `applies_to`.
5. No requirement id is reused or renumbered relative to `main`.
6. Every test-plan row names a fixture or says `none`.
7. The spec is listed in the index in `specs/README.md`.
8. Required sections present, including **Out of scope**.
9. Relative links resolve.

## Quality checks — quote the offending line

- **Untestable requirement.** "MUST handle X correctly" cannot be falsified.
- **Compound requirement.** Two testable claims joined by "and" — split them.
- **Implementation in a spec.** A named library or data structure where a behaviour belongs; that
  is an ADR's job. An interface sketch containing a function body.
- **Missing prohibition.** This domain's requirements are largely negative. If the spec touches
  tokens, does it forbid logging them? If it touches retries, does it forbid retrying
  `invalid_grant`? If it touches storage, does it forbid a plaintext refresh token?
- **A requirement whose test would need `sleep`** — it is phrased wrong; the clock should be
  injected.
- **`applies_to` including an impossible option.**

## Domain checks

Verify against `docs/token-lifecycle.md` and the ADRs:

- `invalid_grant` → mark dead, alert, **zero retries** (including queue-level retries)
- `expires_at` as an absolute instant from the response, never an assumed lifetime
- A refresh response omitting `refresh_token` must not null the stored one
- Granted scopes taken from the response, not the request
- Fail closed when the secret backend is unavailable; one alert, not one per user
- Nothing contradicts an accepted ADR without proposing a superseding one

## Output

Findings worst-first, each with file, line, the concrete defect, and the smallest fix. Then a
verdict: ready for `/tdd-red`, or the specific list of things blocking it. If the spec is sound,
say so and name what you checked — do not invent findings to look thorough.
