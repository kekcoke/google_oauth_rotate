---
id: SPEC-003
title: Consent flow
status: draft
applies_to: [mac, homelab]
depends_on: [SPEC-001, SPEC-004]
---

# SPEC-003 — Consent flow

## Purpose

Obtains the refresh token in the first place, and obtains it again after `invalid_grant` — which
in Testing mode is about weekly. The flow is run by a human at a browser, so its requirements are
as much about clarity and safety as about protocol correctness: consenting the wrong account, or
silently receiving no refresh token, are the two common ways this goes wrong.

`simple/` is excluded: the existing application owns its own consent flow.

## Definitions

- **Authorization code flow with PKCE** — the only flow used here. No implicit flow, no
  password grant.
- **Full consent** — first authorization, or re-consent after `DEAD`. Replaces the stored grant.
- **Incremental consent** — adds scopes to an existing grant with
  `include_granted_scopes=true`.

## Requirements

| ID | Level | Requirement |
|---|---|---|
| REQ-003-01 | MUST | Use the authorization code flow with PKCE (`code_challenge_method=S256`); the verifier is generated per flow and never reused. |
| REQ-003-02 | MUST | Include `access_type=offline`, without which no refresh token is issued. |
| REQ-003-03 | MUST | Include `prompt=consent` on full consent and on re-consent, so a refresh token is returned even for a repeat authorization. |
| REQ-003-04 | MUST | Include `include_granted_scopes=true` on incremental consent, and merge the resulting scope set rather than replacing it. |
| REQ-003-05 | MUST | Generate a cryptographically random, single-use `state`, and reject a callback whose `state` does not match an outstanding flow. |
| REQ-003-06 | MUST | Reject a callback arriving after the flow's expiry window (default 10 minutes). |
| REQ-003-07 | MUST | Fail loudly when the token response contains no refresh token, naming the likely cause, rather than storing an access-token-only record. |
| REQ-003-08 | MUST | Store the granted scope set from the token response, not the requested set. |
| REQ-003-09 | MUST | Derive `user_id` from the ID token's `sub` **only after** the token has passed REQ-003-16, and refuse to overwrite an existing user's record with a different account's grant. |
| REQ-003-10 | MUST | Pass `login_hint` when the target account is known, and display the account the grant was issued for on completion. |
| REQ-003-11 | MUST | Report the granted scope set on completion, so a partially approved consent is visible immediately. |
| REQ-003-12 | MUST | Clear the `DEAD` state and record a re-consent timestamp on successful re-consent. |
| REQ-003-13 | MUST NOT | Log the authorization code, the PKCE verifier, or any token material. |
| REQ-003-14 | MUST | Use the configured redirect URI verbatim, and reject a callback that does not match it: a loopback URI on a **configured fixed port** for `mac/` (default 8765), a fixed HTTPS URL for `homelab/`. An ephemeral port MUST NOT be used unless Google's current documentation has been checked to confirm it does not match the loopback port, and that check is recorded in [`docs/google-cloud-setup.md`](../docs/google-cloud-setup.md). |
| REQ-003-15 | SHOULD | Warn when the granted scope set is narrower than requested, listing the missing scopes. |
| REQ-003-16 | MUST | Verify the ID token before reading any claim from it: signature against Google's published keys, `iss` is a Google issuer, `aud` equals this client id, and the token is unexpired. A token failing any check MUST be rejected and MUST NOT reach the store. |

## Interface sketch

```js
// lib/consent.js

/**
 * @param {Object} opts
 * @param {string[]} opts.scopes
 * @param {'full'|'incremental'} opts.mode
 * @param {string=}  opts.loginHint
 * @returns {{ url: string, state: string, verifier: Secret, expiresAt: Date }}
 */
function begin(opts) {}

/**
 * @throws {StateMismatchError}  unknown or reused `state` (REQ-003-05)
 * @throws {FlowExpiredError}    callback outside the window (REQ-003-06)
 * @throws {NoRefreshTokenError} response lacked a refresh token (REQ-003-07)
 * @throws {AccountMismatchError} `sub` differs from the expected user (REQ-003-09)
 * @throws {IdTokenInvalidError}  signature, iss, aud or exp check failed (REQ-003-16)
 * @returns {Promise<{ userId: string, email: string, grantedScopes: string[], expiresAt: Date }>}
 */
async function complete({ code, state }) {}

/**
 * Verification gate for the ID token. Runs before any claim is read (REQ-003-16).
 * @throws {IdTokenInvalidError} naming which check failed
 * @returns {Promise<{ sub: string, email: string }>}
 */
async function verifyIdToken(idToken) {}
```

## Acceptance criteria

- [ ] A first-run consent yields a stored refresh token and a printed granted-scope set.
- [ ] Re-consent after a forced `invalid_grant` restores service with one command plus a browser.
- [ ] A consent screen where one scope is declined completes, and the narrower granted set is
      reported as a warning rather than passing silently.
- [ ] A replayed callback (same `state` twice) is rejected.
- [ ] Consenting with the wrong Google account is refused, not stored.
- [ ] A forged ID token — valid shape, invalid signature — is refused. `user_id` is the token
      store's primary key, so an unverified `sub` would let a forged token overwrite another
      user's grant.
- [ ] Worker logs from a full flow contain no code, verifier, or token.

## Tests

| Test ID | Covers | What it proves |
|---|---|---|
| T-003-01 | REQ-003-01 | The authorization URL carries `code_challenge` and `code_challenge_method=S256`; two flows produce different verifiers. |
| T-003-02 | REQ-003-02 | The URL carries `access_type=offline`. |
| T-003-03 | REQ-003-03 | Full consent and re-consent carry `prompt=consent`; incremental need not. |
| T-003-04 | REQ-003-04 | Incremental consent carries `include_granted_scopes=true` and the stored scope set becomes the union. |
| T-003-05 | REQ-003-05 | An unknown `state` is rejected; a valid `state` replayed a second time is rejected. |
| T-003-06 | REQ-003-06 | A callback past the flow window is rejected with `FlowExpiredError`. |
| T-003-07 | REQ-003-07 | A token response without `refresh_token` throws `NoRefreshTokenError` and writes nothing. |
| T-003-08 | REQ-003-08 | Given a response granting fewer scopes than requested, the stored set matches the response. |
| T-003-09 | REQ-003-09 | A grant whose ID token `sub` differs from the expected user throws `AccountMismatchError` and leaves the record untouched. |
| T-003-10 | REQ-003-10 | `login_hint` appears in the URL when supplied; the completion result names the account. |
| T-003-11 | REQ-003-11 | The completion result includes the granted scope set. |
| T-003-12 | REQ-003-12 | Re-consenting a `DEAD` record clears the state and sets a re-consent timestamp. |
| T-003-13 | REQ-003-13 | No logger call during a full flow receives the code, the verifier, or a token value. |
| T-003-14 | REQ-003-14 | The authorization URL carries the configured redirect URI verbatim; callbacks on a different host, port, or path are each rejected. |
| T-003-15 | REQ-003-15 | A narrower granted set emits a warning naming exactly the missing scopes. |
| T-003-16 | REQ-003-16 | An ID token with a bad signature, a wrong `iss`, an `aud` for another client, or an expired `exp` is each rejected, and no record is written. |

## Out of scope

- Google Cloud console configuration — [`docs/google-cloud-setup.md`](../docs/google-cloud-setup.md).
- Which scopes to request when — [`spec-004`](spec-004-scope-registry.md).
- The human procedure around re-consent — [`docs/runbooks/re-consent.md`](../docs/runbooks/re-consent.md).
- Any end-user-facing web UI; the `homelab/` callback is an operator endpoint, not a product.

## References

- [ADR 0004](../docs/adr/0004-incremental-authorization.md)
- [ADR 0005](../docs/adr/0005-testing-mode-refresh-token-expiry.md)
- [`docs/oauth-scopes.md`](../docs/oauth-scopes.md)
