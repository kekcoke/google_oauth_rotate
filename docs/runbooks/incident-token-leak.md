# Runbook — Suspected token or credential leak

**Severity:** high. A leaked refresh token lets an attacker mint access tokens for every
granted scope until it is revoked — which for our scope set means reading the mailbox, **sending
mail as the user**, deleting mail, and reading and editing Drive files.

**First action is always revocation at Google.** Cleaning up git history, rotating keys and
writing the postmortem all come after. Only revocation actually stops the attacker.

## Triggers

- A token, client secret, or encryption key appears in a commit, a log file, a screenshot, a
  chat message, a support ticket, or a CI artifact
- Unexpected refresh activity, or refreshes from an unexpected IP in the Vault audit log
- The user reports mail they did not send, or unexplained deletions or Drive changes
- A worker host or container is suspected compromised
- Secret scanning fires in CI

## Immediate actions (first 15 minutes)

1. **Revoke the token at Google.**

   ```text
   POST https://oauth2.googleapis.com/revoke
   Content-Type: application/x-www-form-urlencoded

   token=<refresh_token>
   ```

   Revoking a refresh token invalidates its access tokens too. If you do not hold the value,
   the account owner can revoke the app's access from their Google Account permissions page —
   which kills every grant for the client, for that account.
2. **If the client secret leaked, rotate it now** and accept that all grants for the client die
   ([`vault-unseal-and-rotate.md`](vault-unseal-and-rotate.md), Part 3).
3. **Stop the bleeding.** If a running container or host is the suspected source, stop it. A
   worker that cannot run cannot leak more.
4. **Preserve evidence before cleaning up.** Copy the Vault audit log, worker logs, and the
   offending commit hash somewhere safe. Rewriting history destroys the timeline you will need.
5. **Note the exposure window** — when the secret was created, when it was exposed, when it was
   revoked. Everything downstream depends on this.

## Containment

- Mark affected tokens `DEAD` in the store so no code path tries to use them.
- Revoke the Vault AppRole `secret_id` if the worker identity may be compromised (Part 2 of the
  rotation runbook).
- If an encryption key leaked, treat **every** token it protected as leaked and revoke them all.
- If the leak was in a public repository, assume automated scrapers found it within minutes.
  Speed of revocation matters more than tidiness.

## Assess the damage

Using the exposure window:

- Gmail: check `Sent`, `Trash`, filters and forwarding rules — attackers commonly add a
  forwarding rule or filter to keep access after the token dies. Check
  those explicitly; they survive revocation.
- Drive: review recent file activity and sharing changes for files the app could reach.
- Google Account activity and security events for the affected account.
- Vault audit log for reads that do not correspond to legitimate worker activity.
- Whether other credentials were in the same file, image, or log.

## Recovery

1. Confirm the exposed value is dead — attempt a refresh with it and expect `invalid_grant`.
2. Remove the secret from source control. Rewriting history (`git filter-repo`) is worth doing,
   but it is **not** remediation: the value must already be revoked, because forks, clones,
   caches and forge-side references persist.
3. Re-consent affected accounts: [`re-consent.md`](re-consent.md).
4. Rebuild any image that contained the secret in a layer; deleting a file in a later layer does
   not remove it from the image.
5. Rotate anything else that shared the exposure — keys in the same env file, the same host, the
   same CI secret store.
6. Undo attacker persistence: remove unexpected Gmail filters and forwarding rules, revoke
   unexpected third-party app grants on the account.
7. Verify normal operation, and confirm the leak path is closed.

## Afterwards

- Add a test or a scanner rule for the specific leak path. A leak that could recur unnoticed has
  not been fixed.
- If it was a logging leak, tighten the redaction rules in
  [`spec-007`](../../specs/spec-007-observability.md) and add a test that the offending call
  site cannot log token material.
- If it was a committed file, confirm `.gitignore` covers the pattern and CI secret scanning
  would have caught it.
- Record the timeline, the blast radius, and the fix. Keep it short and blameless; the value is
  in the changed control, not the narrative.
- Notify the account owner. If other people's Google data was reachable, work out the disclosure
  obligation rather than assuming there is none.
