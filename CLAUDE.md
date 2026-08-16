# google_oauth_rotate

Google Workspace OAuth token rotation. Three deployment options for keeping an access token
fresh — without polling Google continuously — plus the specs, tests, and agentic workflow
that build them.

## Current state

**No runtime code exists yet — but `mac/` is now being built.** The active goal is making
[`mac/`](mac/) operational: a container on Docker Desktop refreshing a real Google token. Its
six-step path, and the twelve spec contradictions it resolves along the way, are in
[`mac/IMPLEMENTATION.md`](mac/IMPLEMENTATION.md). Read that before writing anything under `mac/`.

`simple/` and `homelab/` remain specification-only.

Every implementation pass is still driven by a numbered spec and its test plan. If you are asked
to write code for something that has no spec, write the spec first (`/spec-new`).

## The option map

| Directory  | Source | Shape | Token store | Host |
|------------|--------|-------|-------------|------|
| `simple/`  | Option 1 | Alpine + `crond`, 30-min poke of the app's refresh endpoint | Postgres or Redis | any Docker host |
| `mac/`     | Option 2 | Expiry-aware worker: 5-min tick, refresh only within the skew window | SQLite or Postgres | Docker Desktop on macOS, `restart: always` |
| `homelab/` | Option 3 + production | Multi-user SaaS: BullMQ queue worker, refresh-on-demand before API calls, per-user locking | Postgres + Vault/OpenBao | Docker Compose on the homelab box, portable to a cloud VM |

The three options are independent deliverables that share the specs in `specs/`. Changing a
shared spec means revisiting all three test plans.

## The constraint that shapes everything

While the OAuth client is in **Testing** publishing status, Google expires refresh tokens
after **7 days**. Expiry-aware refresh logic does not save you from this. Therefore:

- `invalid_grant` → re-consent is a **first-class, tested path** in every option, never an
  edge case or a `catch` that logs and moves on.
- Any design that assumes an indefinitely valid refresh token is wrong here.
- See `docs/token-lifecycle.md` for the state machine and
  `docs/production-verification.md` for the paths out of Testing mode.

## Scopes

One OAuth client, **incremental authorization**. Requested capabilities: Gmail
read/send/modify, Gmail `watch()` + Pub/Sub push, Drive read/edit, Docs read/edit, Sheets
read/edit. Granted scopes are stored per token and verified before each API call — never
assume a token carries the scope you need. See `docs/oauth-scopes.md`.

## Stack the specs target

- Node.js ≥18, **CommonJS** (`require`/`module.exports`) — no TypeScript, no ESM unless `.mjs`
- Tests: `node:test`, coverage via `c8`
- `google-auth-library` / `googleapis` for OAuth and API calls
- Docker + Docker Compose for all three options
- Postgres, Redis (BullMQ) and Vault/OpenBao in `homelab/`

## Conventions

- **Filenames:** lowercase with hyphens (`refresh-engine.js`, `spec-002-refresh-engine.md`)
- **Commits:** conventional prefixes — `feat`, `fix`, `test`, `docs`, `refactor`, `chore`
- **Tests mirror source, rooted at the option:** modules whose interface sketch lives in a shared
  spec go in `<option>/lib/`, option-specific process code in `<option>/src/`. So
  `mac/lib/refresh-engine.js` → `mac/tests/lib/refresh-engine.test.js` and `mac/src/worker.js` →
  `mac/tests/src/worker.test.js`
- **Requirement IDs:** `REQ-<spec>-<nn>`; test IDs `T-<spec>-<nn>`. Both are permanent —
  never renumber, mark superseded instead.
- **Markdown:** `npx markdownlint-cli '**/*.md' --ignore node_modules` must pass

## Hard security rules

Non-negotiable, enforced by the `token-security-reviewer` agent on any change touching
tokens, logging, or secrets:

1. Never log a refresh token, access token, client secret, or authorization code — not even
   truncated, not even at debug level.
2. Never bake a credential into a Docker image layer or a committed env file.
3. Refresh tokens are encrypted at rest; in `homelab/` they live in Vault/OpenBao, not in a
   plaintext Postgres column.
4. Never store only the access token. If the container dies you must still be able to refresh.
5. Client secrets and tokens reach containers through runtime secret injection, never through
   `docker build --build-arg` and never through `COPY`.

## Workflow

Development loop: `/spec-new` → `/spec-review` → `/tdd-red` → `/tdd-green` → `/verify` →
commit. `/option-status` reports requirement→test→implementation coverage.
Full description in `docs/orchestration-loop.md`.

Tests are written from the test plan **before** implementation. A requirement with no test ID
is an incomplete spec; an implementation with no failing test that preceded it is a process
violation, and the fix is to delete the code and start from the test.
