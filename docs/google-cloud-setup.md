# Google Cloud Setup

One-time console work that must exist before any option can obtain a token. None of it is
automated by this repository; it is a checklist with the decision points called out.

## 1. Project

- Create a dedicated Google Cloud project (for example `oauth-rotate-prod`). Do not reuse a
  project that holds unrelated credentials — blast radius.
- Note the project ID; Pub/Sub topic names are project-scoped.

## 2. Enable APIs

Enable only what is actually used:

- Gmail API
- Google Drive API
- Google Docs API
- Google Sheets API
- Cloud Pub/Sub API — **only** if using `watch()` push (i.e. `homelab/`)

## 3. OAuth consent screen

This is the decision that determines whether refresh tokens survive.

| User type | Requires | Refresh token lifetime | Notes |
|---|---|---|---|
| **External + Testing** | Nothing; add yourself under *Test users* | **7 days** | Where this project starts. Up to 100 test users. |
| **External + Production** | Google verification; CASA assessment for restricted scopes | Long-lived | Weeks of lead time. See [`production-verification.md`](production-verification.md). |
| **Internal** (Workspace only) | A Google Workspace tenant; app owned by that org | Long-lived | No verification needed. The cheapest durable answer **if a Workspace domain is available**. |

Fill in: app name, support email, developer contact, app logo (verification only), homepage,
privacy policy and terms URLs (verification only).

Add the scopes from [`oauth-scopes.md`](oauth-scopes.md). Add only the scopes the first
feature needs — incremental authorization adds the rest later, and every restricted scope on
the consent screen raises the verification bar.

> **Confirm the Workspace question early.** "Internal" needs a Workspace tenant. A consumer
> `gmail.com` account cannot create an internal app, so it is stuck with Testing mode's 7-day
> refresh-token expiry until verification completes. The specs assume 7-day expiry precisely
> because this is unresolved.

## 4. OAuth client credentials

Which client type depends on the option:

| Option | Client type | Redirect URI |
|---|---|---|
| `simple/` | Whatever the existing app already uses | unchanged |
| `mac/` | **Desktop app** | `http://localhost:<port>/oauth2callback` (loopback, ephemeral port) |
| `homelab/` | **Web application** | `https://<your-host>/oauth2callback` — exact match, HTTPS, no wildcards |

Download the client credentials. Store them per [`security-model.md`](security-model.md) —
`homelab/` puts them in Vault; the others use an uncommitted, mode-`600` env file.
`client_secret*.json` is already git-ignored.

Do not use a service account. Service accounts cannot access a consumer Gmail mailbox, and
domain-wide delegation requires a Workspace tenant plus admin consent — a different
architecture from the one specified here.

## 5. Pub/Sub for Gmail push (`homelab/` only)

1. Create a topic, for example `projects/<project-id>/topics/gmail-events`.
2. Grant `roles/pubsub.publisher` on that topic to
   `gmail-api-push@system.gserviceaccount.com`. Gmail's `watch()` call fails without it.
3. Create a **push** subscription pointing at the deployment's HTTPS receiver, with an OIDC
   service-account token so the receiver can authenticate the request.
4. Set an ack deadline and a dead-letter topic; Gmail retries and duplicates are normal.

`simple/` and `mac/` have no public HTTPS ingress, so they use the polling fallback in
[`spec-006-watch-pubsub.md`](../specs/spec-006-watch-pubsub.md) instead. Do not expose a
laptop to the internet to make push work.

**`watch()` expires after 7 days** and must be re-registered — an entirely separate
expiry from the refresh token, with its own scheduled job
([`runbooks/watch-renewal.md`](runbooks/watch-renewal.md)).

## 6. Quotas

Polling intervals must be derived from real figures, not guessed —
[`spec-006`](../specs/spec-006-watch-pubsub.md) REQ-006-13 requires the floor in use to be
recorded here, with the quota figure it comes from.

**Fill this table in during the implementation pass**, from Google's current quota
documentation and the project's own quota page. It is deliberately empty: inventing these
numbers would be worse than admitting they are unknown.

| Limit | Current figure | Where checked | Date checked |
|---|---|---|---|
| Gmail per-user rate limit | *record it* | | |
| Gmail per-project daily quota | *record it* | | |
| `users.history.list` cost in quota units | *record it* | | |
| Max simultaneously valid refresh tokens per client/user | *record it* | | |
| **`POLL_INTERVAL_FLOOR_MS` in use** | *default 60000 until derived* | | |

What is known without a figure:

- Gmail is quota-limited per user per second **and** per project per day; reads cost fewer
  units than mutations, so a polling loop and a mutation loop have different ceilings.
- Token endpoint calls are cheap but not free — the real argument for expiry-aware refresh over
  Option 1's unconditional 30-minute refresh.
- There is a cap on simultaneously valid refresh tokens per OAuth client/user pair; exceeding
  it silently invalidates the oldest. Re-consenting repeatedly during development will
  eventually kill a token you thought was healthy.

## 7. Verification checklist

Before the first `/tdd-green` pass on a consent flow:

- [ ] Project created, APIs enabled
- [ ] Consent screen configured; user type decided and recorded in an ADR
- [ ] Test users added (Testing mode)
- [ ] Client created with the right type and exact redirect URI
- [ ] Credentials stored per the security model, nothing committed
- [ ] Pub/Sub topic + publisher grant + push subscription (`homelab/` only)
- [ ] Quota table above filled in, and `POLL_INTERVAL_FLOOR_MS` derived from it (REQ-006-13)
