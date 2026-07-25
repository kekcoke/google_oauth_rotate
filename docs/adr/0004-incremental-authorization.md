# ADR 0004 — Request scopes incrementally, store what was granted

**Status:** Accepted

## Context

The integration eventually needs Gmail read, send and modify; Drive read/edit; Docs read/edit;
Sheets read/edit; and Gmail push notifications. Asking for all of that on one consent screen is
one dialogue and one code path — but it is also a wall of alarming permissions at first run,
maximum damage if the refresh token leaks, and the heaviest possible verification burden if the
app is ever published.

Google's recommended pattern is incremental authorization: ask for the minimum, add scopes when
a feature needs them, and pass `include_granted_scopes=true` so earlier grants are preserved.

A second, subtler issue: a user can decline individual scopes on the consent screen and the
flow still succeeds. Code that assumes "authorization succeeded, therefore I have my scopes" is
wrong.

## Decision

1. Scopes are requested incrementally, narrowest-first, with `include_granted_scopes=true`.
2. The token store persists `granted_scopes` taken from the **token response**, never from the
   request.
3. Every API adapter declares the scopes it requires; the token provider verifies coverage and
   fails with a named-scope error **before** calling Google.
4. `drive.file` is the default Drive scope. Escalating to full `drive` requires its own ADR
   because it changes the project's compliance posture.

## Consequences

### Good

- Smaller consent screens, and a leaked token grants less.
- Missing-scope failures are deterministic and actionable ("needs `gmail.modify`, granted set
  is `gmail.readonly`") instead of an opaque `403` from Google at runtime.
- In `homelab/`, users can legitimately hold different scope sets — which the design now
  handles rather than being surprised by.

### Bad

- Several consent flows to design and test instead of one, including the merge case.
- The scope registry becomes a real component: a table of feature → required scopes that must
  stay honest, and drifts silently if adapters are added carelessly. Mitigated by a test that
  asserts every adapter is registered.
- Adding scopes to the *client* configuration can invalidate existing grants, so scope changes
  must be treated as re-consent events.

## Alternatives considered

- **Single up-front superset.** One flow, one stored scope set, simplest code. Rejected: worst
  consent experience, largest blast radius, heaviest verification.
- **Two OAuth clients (mail vs files).** Genuinely strong isolation, and worth revisiting if
  the mail and files features ever diverge into separate services. Rejected for now: two
  consent flows, two refresh loops, two token rows per user, and double the operational
  surface for a single integration.

## References

- [`docs/oauth-scopes.md`](../oauth-scopes.md)
- [`specs/spec-004-scope-registry.md`](../../specs/spec-004-scope-registry.md)
