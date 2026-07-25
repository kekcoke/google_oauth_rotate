---
id: SPEC-004
title: Scope registry and incremental authorization
status: draft
applies_to: [mac, homelab]
depends_on: [SPEC-001]
---

# SPEC-004 — Scope registry and incremental authorization

## Purpose

Holds the mapping from feature to required scopes, and enforces it before any Google call. With
incremental authorization a stored token may legitimately lack the scope a new feature needs, and
users of the same deployment may hold different scope sets — so "the token exists, therefore the
call will work" is false. This component makes that failure deterministic and local instead of a
`403` from Google at runtime.

## Definitions

- **Registry** — the declared table of adapter/operation → required scopes.
- **Coverage** — the granted set contains every scope an operation requires.
- **Escalation** — an operation needing a scope not yet granted, resolvable only by incremental
  consent.

## Requirements

| ID | Level | Requirement |
|---|---|---|
| REQ-004-01 | MUST | Declare required scopes for every API operation in one registry, not scattered at call sites. |
| REQ-004-02 | MUST | Verify coverage before issuing a Google API call, and fail with an error naming the missing scopes and the granted set when it is insufficient. |
| REQ-004-03 | MUST | Compare scopes exactly as strings; MUST NOT infer that one scope implies another. |
| REQ-004-04 | MUST NOT | Degrade to a narrower behaviour when a scope is missing — no silent read-only fallback for a write operation. |
| REQ-004-05 | MUST | Merge newly granted scopes into the stored set on incremental consent, deduplicated. |
| REQ-004-06 | MUST | Support per-user scope sets: two users of one deployment may hold different sets, and coverage is evaluated per user. |
| REQ-004-07 | MUST | Fail startup if any registered adapter operation references a scope absent from the known-scope catalogue (catches typos in scope URLs). |
| REQ-004-08 | MUST | Fail startup if an adapter operation exists with no registry entry, so a new adapter cannot bypass verification. |
| REQ-004-09 | MUST | Default Drive access to `drive.file`; requesting full `drive` MUST require an explicit, separately named configuration flag. |
| REQ-004-10 | SHOULD | Produce, for a given user, the exact scope list an incremental consent needs to unblock a requested operation. |
| REQ-004-11 | SHOULD | Treat a Google `403 insufficient_permission` as a registry defect and log it as such — the check should have caught it first. |

## Interface sketch

```js
// lib/scope-registry.js

/** Known-scope catalogue; typo protection (REQ-004-07). */
const SCOPES = {
  GMAIL_READONLY: 'https://www.googleapis.com/auth/gmail.readonly',
  GMAIL_SEND:     'https://www.googleapis.com/auth/gmail.send',
  GMAIL_MODIFY:   'https://www.googleapis.com/auth/gmail.modify',
  DRIVE_FILE:     'https://www.googleapis.com/auth/drive.file',
  DRIVE_FULL:     'https://www.googleapis.com/auth/drive',
  DOCUMENTS:      'https://www.googleapis.com/auth/documents',
  SPREADSHEETS:   'https://www.googleapis.com/auth/spreadsheets',
};

/** operation id -> required scopes (REQ-004-01) */
const REGISTRY = {
  'gmail.listMessages': [SCOPES.GMAIL_READONLY],
  'gmail.sendMessage':  [SCOPES.GMAIL_SEND],
  'gmail.modifyLabels': [SCOPES.GMAIL_MODIFY],
  'docs.updateDoc':     [SCOPES.DRIVE_FILE, SCOPES.DOCUMENTS],
  // …
};

/** @returns {{ covered: boolean, missing: string[] }} */
function checkCoverage(operationId, grantedScopes) {}

/** @throws {ScopeInsufficientError} naming missing and granted sets (REQ-004-02) */
function assertCoverage(operationId, grantedScopes) {}

/** @returns {string[]} scopes to request to unblock `operationId` (REQ-004-10) */
function scopesToEscalate(operationId, grantedScopes) {}

/** Startup validation (REQ-004-07, REQ-004-08). @throws {RegistryError} */
function validateRegistry(adapterOperationIds) {}
```

## Acceptance criteria

- [ ] Calling a send operation with a read-only grant fails locally, before any network call,
      naming `gmail.send`.
- [ ] Adding a new adapter operation without a registry entry breaks startup.
- [ ] A misspelled scope URL breaks startup.
- [ ] Two users with different grants get different coverage answers for the same operation.
- [ ] Full `drive` cannot be requested without the explicit flag.
- [ ] `scopesToEscalate` output can be handed straight to the incremental consent flow.

## Tests

| Test ID | Covers | What it proves |
|---|---|---|
| T-004-01 | REQ-004-01 | Every adapter operation resolves to a registry entry. |
| T-004-02 | REQ-004-02 | Insufficient coverage throws `ScopeInsufficientError` with the missing scopes named, and no HTTP request is made. |
| T-004-03 | REQ-004-03 | Holding `gmail.modify` does not satisfy an operation requiring `gmail.readonly` unless it is listed explicitly. |
| T-004-04 | REQ-004-04 | A write operation with only a read grant throws; it does not perform a read instead. |
| T-004-05 | REQ-004-05 | Merging grants yields the deduplicated union, order-insensitive. |
| T-004-06 | REQ-004-06 | Two users with different granted sets yield different coverage results for one operation. |
| T-004-07 | REQ-004-07 | A registry entry with an unknown scope string fails `validateRegistry`. |
| T-004-08 | REQ-004-08 | An adapter operation missing from the registry fails `validateRegistry`. |
| T-004-09 | REQ-004-09 | Full `drive` is absent from requested scopes unless the explicit flag is set. |
| T-004-10 | REQ-004-10 | `scopesToEscalate` returns exactly the missing scopes, and an empty array when coverage is complete. |
| T-004-11 | REQ-004-11 | A faked `403 insufficient_permission` is logged as a registry defect with the operation id. |

## Out of scope

- Which scopes Google classifies as sensitive or restricted, and the verification consequences —
  [`docs/oauth-scopes.md`](../docs/oauth-scopes.md).
- Running the consent flow — [`spec-003`](spec-003-consent-flow.md).
- Adapter behaviour beyond scope declaration — [`spec-005`](spec-005-api-adapters.md).

## References

- [ADR 0004](../docs/adr/0004-incremental-authorization.md)
- [`docs/oauth-scopes.md`](../docs/oauth-scopes.md)
