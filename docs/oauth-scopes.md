# OAuth Scopes

One OAuth client covers Gmail, Drive, Docs and Sheets. Scopes are requested **incrementally**
— never all at once — and the set actually granted is stored per token and checked before
every call. Requirements live in
[`spec-004-scope-registry.md`](../specs/spec-004-scope-registry.md).

> **Verify before you build.** Google moves scopes between the non-sensitive / sensitive /
> restricted tiers, and the tier decides whether verification and a CASA security assessment
> apply. The "Tier" column below is a planning assumption, not a fact to trust. Confirm each
> row against Google's current OAuth scope documentation during the implementation pass and
> correct this table in the same commit.

## Scope inventory

| Capability | Scope | Tier (verify) | Why we want it |
|---|---|---|---|
| Read mail | `https://www.googleapis.com/auth/gmail.readonly` | restricted | List and fetch messages, threads, history |
| Send mail | `https://www.googleapis.com/auth/gmail.send` | sensitive | Outbound mail only, no read implied |
| Label / archive / delete | `https://www.googleapis.com/auth/gmail.modify` | restricted | Mutating mailbox operations |
| Push notifications | *(no extra OAuth scope)* — `watch()` needs `gmail.readonly` or `gmail.modify`, plus a Pub/Sub topic granting publish rights to `gmail-api-push@system.gserviceaccount.com` | — | Real-time mailbox change events |
| Files created/opened by this app | `https://www.googleapis.com/auth/drive.file` | non-sensitive | Read and edit the app's own files |
| Whole Drive | `https://www.googleapis.com/auth/drive` | **restricted** | Only if the app must touch files it did not create |
| Docs read/edit | `https://www.googleapis.com/auth/documents` | sensitive | Create and edit Google Docs |
| Sheets read/edit | `https://www.googleapis.com/auth/spreadsheets` | sensitive | Create and edit Google Sheets |
| Identity | `openid`, `email`, `profile` | non-sensitive | Stable `sub` as the token store's `user_id` |

### Prefer `drive.file` over `drive`

Full `drive` is a restricted scope: it pulls the client into Google's verification process
and, for a production public app, an annual CASA security assessment. `drive.file` grants
access only to files the app created or the user explicitly opened with it, is not restricted,
and is sufficient for the common "the automation makes and maintains its own docs and sheets"
case.

**Default to `drive.file`.** Escalating to full `drive` is an architectural decision that
needs an ADR recording the specific operation that forced it, because it changes the
compliance posture of the entire project.

Note that `documents` and `spreadsheets` govern *content* access to Docs and Sheets while
Drive scopes govern *file* access. Editing a Doc created by the app needs `drive.file` **and**
`documents`.

## Incremental authorization

The first consent asks for identity plus the narrowest scope the first feature needs. Later
features trigger an incremental consent that adds scopes, carrying
`include_granted_scopes=true` so the previously granted set is preserved rather than replaced.

```text
first run        -> openid email profile gmail.readonly
adds sending     -> + gmail.send
adds filing      -> + gmail.modify
adds doc output  -> + drive.file documents
adds sheets      -> + spreadsheets
```

Rules:

1. **Store what was granted, not what was asked.** The token response's `scope` field is the
   truth. A user can deny individual scopes on the consent screen and the flow still succeeds.
2. **Check before calling.** Every API adapter declares its required scopes; the token
   provider rejects the call with a named-scope error if `granted_scopes` does not cover them.
   Do not learn this from a `403` from Google.
3. **Never silently downgrade.** If a feature needs `gmail.modify` and only `gmail.readonly`
   was granted, fail loudly and point at the incremental consent command — do not fall back to
   read-only behaviour and pretend it worked.
4. **Scope changes can kill the refresh token.** Adding scopes to the *client* configuration,
   as opposed to requesting more at consent time, can invalidate existing grants. Treat it as
   a re-consent event.

## Consent request parameters

| Parameter | Value | Reason |
|---|---|---|
| `access_type` | `offline` | Without it there is no refresh token at all |
| `prompt` | `consent` on first authorization and on re-consent | Guarantees a refresh token is returned; Google may omit it on repeat authorizations |
| `include_granted_scopes` | `true` | Incremental authorization preserves the existing grant |
| `login_hint` | the target account | Avoids consenting the wrong account on a machine with several signed in |
| PKCE (`code_challenge`) | S256 | Required for the installed-app style flow used by `simple/` and `mac/` |
| `state` | random, single-use | CSRF protection on the `homelab/` web callback |

## Least privilege by option

| Option | Scopes it should hold |
|---|---|
| `simple/` | Whatever the existing app already uses — this option does not own the consent flow |
| `mac/` | Only the personal automation's scopes; typically `gmail.readonly` + `drive.file` + `documents`/`spreadsheets` as needed |
| `homelab/` | Per-user, superset across features, but each stored grant reflects only what that user approved |

Two users of the `homelab/` deployment can legitimately hold different scope sets. Code that
assumes a uniform scope set across users is wrong.
