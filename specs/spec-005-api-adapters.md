---
id: SPEC-005
title: Google API adapters
status: draft
applies_to: [mac, homelab]
depends_on: [SPEC-002, SPEC-004, SPEC-009]
---

# SPEC-005 — Google API adapters

## Purpose

The thin layer between application code and the Gmail, Drive, Docs and Sheets APIs. Its job is
not to wrap every endpoint — it is to guarantee that **every** Google call goes through the same
gate: a valid token, verified scopes, classified errors, and no mutation without an explicit,
auditable intent. An adapter is the only place allowed to hold a Google API client.

## Definitions

- **Adapter** — a module for one API surface (`gmail`, `drive`, `docs`, `sheets`).
- **Operation** — one named, registered call (`gmail.sendMessage`), the unit the scope registry
  keys on.
- **Mutating operation** — anything that changes state in the user's account.
- **Dry run** — a mutating operation executed to the point of describing exactly what it would
  change, without changing it.

## Requirements

| ID | Level | Requirement |
|---|---|---|
| REQ-005-01 | MUST | Obtain the access token through the refresh engine's `getValidToken` immediately before the call; MUST NOT cache a token across calls in adapter state. |
| REQ-005-02 | MUST | Assert scope coverage via [`spec-004`](spec-004-scope-registry.md) before the request. |
| REQ-005-03 | MUST | Route every failure through the error taxonomy in [`spec-009`](spec-009-error-taxonomy.md); MUST NOT expose a raw Google error to callers. |
| REQ-005-04 | MUST | Support `dryRun` on every mutating operation, returning a description of the intended change and performing none. |
| REQ-005-05 | MUST | Be idempotent, or documented as not idempotent with the reason, for every operation invoked by a retryable job. |
| REQ-005-06 | MUST | Require an explicit, non-default argument to perform a destructive operation (permanent delete, permission removal); a caller MUST NOT be able to destroy data by omitting a flag. |
| REQ-005-07 | MUST | Handle pagination completely, or expose the page token to the caller; MUST NOT silently return a first page as if it were the whole result. |
| REQ-005-08 | MUST | Log operation, `user_id`, outcome and duration — never message bodies, file contents, or token material. |
| REQ-005-09 | MUST | Apply a request timeout, and classify a timeout as transient. |
| REQ-005-10 | MUST | Register every operation in the scope registry, enforced at startup (REQ-004-08). |
| REQ-005-11 | SHOULD | Prefer `drive.file`-compatible calls, so the default Drive scope suffices. |
| REQ-005-12 | SHOULD | Batch where the API supports it, to stay inside per-user rate limits. |
| REQ-005-13 | MUST | Treat Gmail message and Drive file content as user data: not written to logs, not persisted outside its intended destination, redacted in error reports. |

## Interface sketch

```js
// lib/adapters/gmail.js
/**
 * @param {Object} deps  { getValidToken, registry, logger, http }
 */
function createGmailAdapter(deps) {
  return {
    /** @returns {Promise<{messages: object[], nextPageToken?: string}>} */
    listMessages(userId, { query, pageToken, maxResults }) {},
    /** @returns {Promise<object>} */
    getMessage(userId, messageId) {},
    /** @param {{dryRun?: boolean}} opts */
    sendMessage(userId, message, opts) {},
    /** @param {{dryRun?: boolean}} opts */
    modifyLabels(userId, messageId, { add, remove }, opts) {},
    /** @param {{dryRun?: boolean, confirmPermanent: true}} opts (REQ-005-06) */
    deleteMessage(userId, messageId, opts) {},
  };
}

// lib/adapters/drive.js   listFiles, getFile, createFile, updateFile
// lib/adapters/docs.js    getDoc, batchUpdate
// lib/adapters/sheets.js  getValues, updateValues, appendValues
```

Every adapter is constructed with injected dependencies; none reads configuration or
credentials directly.

## Acceptance criteria

- [ ] No adapter holds a token between calls — verified by inspection and by a test that changes
      the token between two calls.
- [ ] Every mutating operation supports `dryRun` and is covered by a dry-run test.
- [ ] A permanent delete cannot be triggered without the explicit confirmation argument.
- [ ] A faked `429` produces a taxonomy-classified transient error, not a raw Google error.
- [ ] A two-page list result returns both pages, or exposes the page token.
- [ ] Logs from a full adapter test run contain no message bodies or file contents.

## Tests

| Test ID | Covers | What it proves |
|---|---|---|
| T-005-01 | REQ-005-01 | `getValidToken` is called once per operation; a token rotated between two calls is picked up by the second. |
| T-005-02 | REQ-005-02 | An operation with insufficient scope throws before any HTTP request is made. |
| T-005-03 | REQ-005-03 | Faked Google errors (`401`, `403`, `429`, `500`, network reset) each surface as their taxonomy class. |
| T-005-04 | REQ-005-04 | Every mutating operation with `dryRun: true` returns a change description and issues no mutating request. |
| T-005-05 | REQ-005-05 | Invoking a retryable operation twice with the same input produces one net effect, or the operation is documented non-idempotent. |
| T-005-06 | REQ-005-06 | `deleteMessage` without `confirmPermanent` throws; with it, the request is issued. |
| T-005-07 | REQ-005-07 | A faked two-page response yields all items, or a `nextPageToken` the caller must follow. |
| T-005-08 | REQ-005-08 | Logs contain operation, user and outcome, and no body, content, or token material. |
| T-005-09 | REQ-005-09 | A hung request aborts at the timeout and is classified transient. |
| T-005-10 | REQ-005-10 | Every exported operation appears in the scope registry. |
| T-005-11 | REQ-005-11 | Drive operations succeed with only `drive.file` granted, for the app's own files. |
| T-005-12 | REQ-005-12 | A batched call path issues one request where the API supports batching. |
| T-005-13 | REQ-005-13 | An error thrown mid-operation does not carry message body or file content in its message or properties. |

## Out of scope

- Business logic that uses these adapters — this project specifies the plumbing.
- Push notification handling — [`spec-006`](spec-006-watch-pubsub.md).
- Google API quota tuning beyond honouring `429` — [`spec-009`](spec-009-error-taxonomy.md).
- Any adapter for an API not listed in [`docs/oauth-scopes.md`](../docs/oauth-scopes.md).

## References

- [`docs/oauth-scopes.md`](../docs/oauth-scopes.md)
- [`specs/spec-004-scope-registry.md`](spec-004-scope-registry.md)
