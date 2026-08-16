# Getting Out of Testing Mode

The project is specified to operate correctly in **Testing** publishing status, where Google
expires refresh tokens after 7 days. That is a survivable annoyance for personal automation
and an unacceptable one for anything else. There are two exits.

## Exit A — Google Workspace internal app (fast, if available)

If the app is owned by a Google Workspace organisation and its user type is **Internal**:

- No verification, no CASA assessment, no brand review
- Refresh tokens are long-lived
- Restricted scopes are usable immediately
- Only users in that Workspace can consent

Steps:

1. Confirm a Workspace tenant exists and you can administer its Cloud organisation.
2. Create the Cloud project **inside that organisation** — a project owned by a personal
   account cannot be switched to Internal later without moving it.
3. Set the consent screen's user type to Internal.
4. Re-consent once. Existing Testing-mode tokens are not migrated.
5. Set `OAUTH_PUBLISHING_STATUS=internal`, which turns off the day-6 age alerts
   (REQ-002-13, REQ-007-08). Then delete the re-consent reminder from the maintenance schedule
   and record the change in an ADR — the flag silences the alerts, but the scheduled job is dead
   weight and should go.

**This is the recommended exit** where a Workspace domain is available, because it removes
the 7-day problem in an afternoon rather than a quarter. It is unavailable to consumer
`@gmail.com` accounts.

## Exit B — External + Production with verification

Required if outside users must consent, or no Workspace tenant exists.

### What triggers what

| Scope tier | Requirement |
|---|---|
| Non-sensitive (`drive.file`, `openid`, `email`, `profile`) | Brand verification only |
| Sensitive (`documents`, `spreadsheets`, `gmail.send`) | Google review of scope justification |
| **Restricted** (`gmail.readonly`, `gmail.modify`, full `drive`) | Review **plus** an annual third-party CASA security assessment, and adherence to the Limited Use requirements |

Our scope set includes restricted scopes, so Exit B means CASA. Budget weeks, not days, and a
real cost for the assessment.

### Readiness checklist

- [ ] **Domain ownership** verified in Search Console for the homepage domain
- [ ] **Homepage** on that domain, publicly reachable, explaining what the app does
- [ ] **Privacy policy** on the same domain, describing exactly what Google user data is
      collected, how it is used, retained, and deleted, plus the Limited Use disclosure
- [ ] **Terms of service** (if applicable)
- [ ] **App name, logo, support email** consistent everywhere; no Google branding
- [ ] **Scope justification** per scope: the specific feature that needs it, why a narrower
      scope will not do. Reviewers reject vague answers.
- [ ] **Demo video** (unlisted is fine) showing the consent screen, the OAuth client ID
      visible in the URL, and each requested scope actually being used in the product
- [ ] **Narrowest possible scopes** — swap full `drive` for `drive.file` before applying if at
      all possible; it removes a restricted scope from the request
- [ ] **Limited Use compliance:** no transferring Google user data to third parties except as
      allowed, no human reading of data except as allowed, no ads, no selling
- [ ] **Security posture for CASA:** secrets management (this repo's Vault design helps),
      encryption in transit and at rest, access control, dependency patching, incident response
- [ ] **Data deletion path** for a user who revokes — implemented and demonstrable
- [ ] All of the above true of the *deployed* app, not just the spec

### Sequence

1. Reduce scopes to the minimum that still ships the product.
2. Publish homepage, privacy policy, terms.
3. Verify domain ownership.
4. Record the demo video.
5. Submit for verification from the consent screen page.
6. Answer reviewer follow-ups — expect at least one round.
7. Complete the CASA assessment for restricted scopes.
8. Publish to Production, re-consent, and confirm refresh tokens outlive 7 days before
   removing the re-consent reminder.

## Until then

The maintenance loop assumes Testing mode:

- Daily `token-health` check reporting each token's age and expiry
- Warning at **day 6** of refresh-token age, before day 7 breaks it
- `/reconsent` as a routine, documented operation, not an incident
- Alerting on `invalid_grant` that names the affected account and links the runbook

See [`runbooks/re-consent.md`](runbooks/re-consent.md) and
[`orchestration-loop.md`](orchestration-loop.md).

## Recording the decision

Whichever exit is taken, update
[`adr/0005-testing-mode-refresh-token-expiry.md`](adr/0005-testing-mode-refresh-token-expiry.md)
with a *Superseded by* note rather than editing its history, and revisit
[`spec-002-refresh-engine.md`](../specs/spec-002-refresh-engine.md) — some tests
(day-6 warning, weekly re-consent) become obsolete and should be deleted, not left to rot.
