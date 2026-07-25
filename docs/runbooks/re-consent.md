# Runbook — Re-consent

**When:** a token is in state `DEAD` (`invalid_grant` on refresh), or the day-6 age warning
fired, or scopes were added to the OAuth client configuration.

**Expected frequency:** roughly weekly while the OAuth client is in Testing status. This is
routine maintenance, not an incident. See
[ADR 0005](../adr/0005-testing-mode-refresh-token-expiry.md).

**Time to resolve:** a few minutes, most of it waiting for a browser.

## Symptoms

- Alert: `token_state=DEAD reason=invalid_grant user_id=<...>`
- `/token-health` reports `refresh_token_age_days >= 7`, or a `DEAD` token
- API calls failing with `401` after a refresh attempt that did not succeed

## Confirm the cause first

`invalid_grant` has several causes and they lead to different follow-ups:

| Cause | How to tell | Follow-up |
|---|---|---|
| Testing-mode 7-day expiry | Token age ≈ 7 days, no other change | Re-consent; consider an exit ([`production-verification.md`](../production-verification.md)) |
| User revoked access | Token age < 7 days; user confirms, or the grant is absent from their Google Account permissions page | Re-consent, and ask why |
| Password change | User confirms a recent change | Re-consent |
| Client secret rotated | Rotation happened recently | Re-consent every affected account |
| Client scope configuration changed | A scope was added/removed in the console | Re-consent every affected account |
| Too many live refresh tokens for this client/user | Many recent re-consents during development | Re-consent; stop re-consenting in a loop |

Do not skip this step. Re-consenting a revoked grant without asking why can walk straight back
into a security event.

## Procedure

1. **Check whether anything else is broken.** A sealed Vault or an unreachable database
   produces alerts that look adjacent; fix those first, since re-consent has to write a token.
2. **Run the command.**

   ```text
   /reconsent --option <simple|mac|homelab> --user <user_id>
   ```

   It prints the authorization URL with `access_type=offline`, `prompt=consent`,
   `include_granted_scopes=true`, and the scopes currently recorded for that user.
3. **Open the URL as the correct Google account.** `login_hint` should pre-select it; verify
   the account shown on the consent screen before approving. Consenting with the wrong account
   creates a valid token for the wrong mailbox — a confusing failure to unpick later.
4. **Approve every scope shown.** A partially approved consent succeeds but leaves the token in
   `SCOPE_INSUFFICIENT` for some features.
5. **Confirm the write.** The command should report the new expiry, the granted scope set, and
   a re-consent timestamp. Granted scopes come from the token response — if the set is smaller
   than expected, something was declined at step 4.
6. **Verify with a real call.** `/token-health --option <...> --user <...>` and one read-only
   API call. A stored token that has never been used is not yet proof of anything.
7. **Clear the alert** and note the cause in the maintenance log.

## Per-option specifics

| Option | Notes |
|---|---|
| `simple/` | The app owns the consent flow; this option only pokes its refresh endpoint. Re-consent happens in the app, then verify the worker's next tick succeeds. |
| `mac/` | Desktop-app client on a loopback redirect. The container has no browser: the flow is started on the host, and the resulting token is written to the store the container shares. Confirm the container picked it up on its next tick rather than assuming. |
| `homelab/` | Web-app client on the deployment's HTTPS callback. The new refresh token is written to Vault at `secret/oauth/<user_id>/refresh`; Postgres keeps only the path. Confirm the Vault write succeeded, and confirm the per-user lock is not still held from the failed refresh. |

## If it fails

- **Consent screen refuses with an app-not-verified block:** for a Testing-mode app, the
  account must be listed under *Test users*. Add it in the console.
- **No refresh token in the response:** `prompt=consent` was missing, or `access_type=offline`
  was not set. Google omits the refresh token on repeat authorizations without them.
- **`redirect_uri_mismatch`:** the client's registered URI must match exactly, including
  scheme, host, port and path.
- **Token written but immediately dead again:** suspect a client-secret mismatch between what
  the worker holds and what the console has.

## Prevention

- Keep the day-6 warning until an exit from Testing mode is taken.
- Do not re-consent repeatedly for testing; each grant consumes a slot and eventually evicts
  older tokens.
- Take an exit. Weekly manual work forever is the actual problem here, and
  [`production-verification.md`](../production-verification.md) documents both ways out.
