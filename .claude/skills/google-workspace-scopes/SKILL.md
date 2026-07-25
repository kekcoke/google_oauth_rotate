---
name: google-workspace-scopes
description: Gmail, Drive, Docs and Sheets scope selection for this project — incremental authorization, restricted vs sensitive tiers, verification consequences, and the consent parameters that matter. Use when choosing scopes, building a consent flow, or debugging a 403.
---

# Google Workspace scopes

## Verify before you trust

Google moves scopes between the non-sensitive, sensitive and restricted tiers, and the tier decides
whether verification and a CASA assessment apply. `docs/oauth-scopes.md` carries a planning
assumption, not a fact. **Fetch the current scope documentation** when the tier matters, and correct
the doc in the same change.

## The scope set

| Capability | Scope | Notes |
|---|---|---|
| Read mail | `gmail.readonly` | Restricted |
| Send mail | `gmail.send` | No read implied |
| Label / archive / delete | `gmail.modify` | Restricted; mutating |
| Push notifications | *(no extra scope)* | `watch()` needs `gmail.readonly` or `gmail.modify`, plus a Pub/Sub topic granting publish to `gmail-api-push@system.gserviceaccount.com` |
| App's own files | `drive.file` | **The default.** Not restricted |
| Whole Drive | `drive` | Restricted → CASA. Needs an ADR to adopt |
| Docs content | `documents` | Sensitive |
| Sheets content | `spreadsheets` | Sensitive |
| Identity | `openid`, `email`, `profile` | `sub` is the token store's `user_id` |

**Editing an app-created Doc needs `drive.file` AND `documents`.** Drive scopes govern the file;
Docs and Sheets scopes govern the content. Missing one produces a `403` that looks like a bug in
the wrong layer.

## Prefer `drive.file`

Full `drive` is restricted: verification plus an annual third-party security assessment for a
public app. `drive.file` covers files the app created or the user explicitly opened with it, and is
not restricted. Escalating changes the compliance posture of the whole project — it is an ADR
decision, not a convenience.

## Incremental authorization

Ask for the minimum first; add scopes when a feature needs them.

```text
first run        -> openid email profile gmail.readonly
adds sending     -> + gmail.send
adds filing      -> + gmail.modify
adds doc output  -> + drive.file documents
adds sheets      -> + spreadsheets
```

Four rules:

1. **Store what was granted, not what was asked.** The token response's `scope` field is the truth.
   A user can decline individual scopes and the flow still succeeds.
2. **Check coverage before calling.** Every operation declares its scopes; the check fails locally,
   naming the missing scope. Learning this from a Google `403` is too late.
3. **Never silently downgrade.** A write operation with only a read grant fails loudly; it does not
   quietly read instead.
4. **Per-user scope sets differ.** In a multi-user deployment, two users legitimately hold different
   grants. Code assuming a uniform set is wrong.

## Consent parameters

| Parameter | Value | Why |
|---|---|---|
| `access_type` | `offline` | Without it there is **no refresh token at all** |
| `prompt` | `consent` on first authorization and re-consent | Google may otherwise omit the refresh token on a repeat authorization |
| `include_granted_scopes` | `true` on incremental consent | Preserves the existing grant instead of replacing it |
| `login_hint` | the target account | Stops consenting the wrong account on a machine with several signed in |
| PKCE | S256, fresh verifier per flow | Required for the installed-app flow |
| `state` | random, single-use | CSRF protection on a web callback |

## Debugging a 403

`403` has several meanings and they route to different classes:

| Reason | Meaning | Action |
|---|---|---|
| `insufficient_permission` / scope-insufficient | Scope not granted | Registry defect — the local check should have caught it. Incremental consent |
| Rate-limit reason | Too fast | Transient; backoff |
| Quota exceeded | Daily or project quota | Transient, but fix the call pattern |
| `403` on Drive while holding `drive.file` | The file was not created or opened by this app | Expected behaviour of `drive.file`, not a bug |

A scope change on the **client configuration** (as opposed to requesting more at consent time) can
invalidate existing grants — treat it as a re-consent event.

## Verification consequences

Restricted scopes in our set mean that publishing to Production requires Google review plus an
annual CASA assessment. The cheap alternative is a Workspace-internal app. Until one of those
happens, Testing mode means refresh tokens die every 7 days. Details:
`docs/production-verification.md`.

## References

- `docs/oauth-scopes.md` — the inventory and per-option least privilege
- `specs/spec-004-scope-registry.md` — the enforcement requirements
- `specs/spec-003-consent-flow.md` — the flow requirements
