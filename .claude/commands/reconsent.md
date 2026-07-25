---
description: Walk through re-consent for a dead or expiring grant, following the runbook
argument-hint: --option simple|mac|homelab [--user <user_id>]
---

Re-consent: $ARGUMENTS

Follow `docs/runbooks/re-consent.md`. Expected roughly weekly while the OAuth client is in Testing
status — this is routine maintenance, not an incident.

## Before generating a URL

**Establish why the grant died.** `invalid_grant` has several causes and they lead to different
follow-ups:

| Cause | Tell | Follow-up |
|---|---|---|
| Testing-mode 7-day expiry | Token age ≈ 7 days, nothing else changed | Re-consent; consider an exit |
| User revoked access | Age < 7 days | Re-consent, **and ask why** |
| Password change | User confirms | Re-consent |
| Client secret rotated | Rotation happened recently | Re-consent every affected account |
| Client scope config changed | Console change | Re-consent every affected account |
| Too many live refresh tokens | Many recent re-consents in development | Re-consent; stop looping |

Do not skip this. Re-consenting a revoked grant without asking why can walk straight back into a
security event.

Then confirm nothing else is broken — a sealed Vault or an unreachable database will make the
re-consent write fail, and those symptoms look adjacent.

## Steps

1. Report the current state: `state`, token age, granted scopes, last successful refresh.
2. Generate the authorization URL with `access_type=offline`, `prompt=consent`,
   `include_granted_scopes=true`, `login_hint` for the target account, and the scopes currently
   recorded for that user. State the scopes you are requesting before printing the URL.
3. Have the human complete it in a browser. Tell them to **check the account shown on the consent
   screen** — consenting the wrong account creates a valid token for the wrong mailbox, which is
   confusing to unpick later.
4. On completion, report the account, the **granted** scope set, the new expiry, and the re-consent
   timestamp. A granted set narrower than requested means something was declined; say which scopes.
5. Verify with one real read-only API call. A stored token that has never been used proves nothing.
6. For `homelab/`, confirm the refresh token reached Vault and that Postgres holds only a path, and
   that no stale per-user lock is still held from the failed refresh.
7. For `mac/`, confirm the container picked the record up on its next tick — the consent helper runs
   on the host, and a record written to the wrong store looks identical to one never written.

## Per option

- `simple/` — the application owns the consent flow. Drive it there, then confirm the worker's next
  tick succeeds.
- `mac/` — desktop client, loopback redirect, helper runs on the host.
- `homelab/` — web client on the HTTPS callback.

Never print a token value. Report the account, scopes, and expiry only.
