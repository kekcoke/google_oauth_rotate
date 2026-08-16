# `mac/` — Implementation plan

Context for any agent or human working toward one goal: **make `mac/` actually run on this Mac
and refresh a real Google OAuth token.**

Read this before `/tdd-red`, `/tdd-green`, `/loop-dev` or `/verify` on anything under `mac/`.
It does not replace the specs — it records the decisions the specs left open, and the order in
which the work is done.

## Definition of done

A `gmail-worker` container on Docker Desktop that:

- ticks every 5 minutes and refreshes **only** inside the 10-minute skew window,
- holds a real, encrypted refresh token on a named volume,
- survives `docker compose restart` with no state loss,
- re-evaluates immediately after a lid-close clock jump rather than waiting for the next tick,
- reports health on `127.0.0.1:8080` and is unreachable from the LAN,
- refuses to start when the encryption key or the store is missing.

The milestone is **step 3**. Steps 4–6 take it from working to trustworthy.

## Current state

Specification-only. No runtime code exists anywhere in the repository — no `package.json`,
`Dockerfile`, `docker-compose.yml`, `lib/`, `src/`, `tests/` or eslint config. All 15 specs are
`status: draft`; all 7 ADRs are `Accepted`. There is no git remote, so CI has never run.

Scope for `mac/`: **131 requirements, 114 of them MUST, 138 test IDs** (137 planned, plus
T-008-11 reinstated in step 1). Roughly 27 are integration level, the rest unit; zero e2e in CI.

## Standing decisions

Twelve contradictions and gaps were found across the specs and docs. Each is resolved here so it
is not rediscovered mid-build. Where a decision changes a spec, step 1 makes that change.

### B1 — Code layout: `mac/lib/` and `mac/src/`, both

`mac/` is a self-contained npm package. Modules whose interface sketch lives in a **shared** spec
go in `mac/lib/`; option-specific process code goes in `mac/src/`, exactly as SPEC-200 names it.

```text
mac/lib/     token-store.js  secret.js  refresh-engine.js  consent.js  scope-registry.js
             logger.js  health.js  config.js  errors.js  history-sync.js  oauth-client.js
             secrets/{index,envelope}.js   adapters/{gmail,drive,docs,sheets}.js
             stores/sqlite.js
mac/src/     worker.js  instance-guard.js  index.js  consent-cli.js
mac/tests/   lib/*.test.js   src/*.test.js
mac/scripts/ up.sh  consent.sh
```

This needs **zero spec edits** — the sketches say `lib/token-store.js` and `src/worker.js`, and
both resolve verbatim. Root `CLAUDE.md` already states the three options are independent
deliverables that share the *specs*, not the code, so a per-option package is on-doctrine. No npm
workspaces and no repo-root `src/`: `homelab/` does not exist, and extracting a shared package
before there is a second consumer buys nothing. Three stale
`src/refresh-engine.js → tests/refresh-engine.test.js` claims are corrected in step 2.

### B2 — Consent runs in a one-shot container, not on the host

`mac/docs/deploy.md` currently says `node scripts/consent.js` runs on the macOS host and writes
"to the same volume the container uses". **A macOS host process cannot write a Docker named
volume** — it lives inside the Linux VM — and `sqlite:///data/tokens.db` is a container-absolute
path. This is the largest blocker in the option, and `deploy.md` lists its symptom as a
troubleshooting row rather than fixing it.

Resolution: a second Compose service, same image, behind a profile.

```yaml
consent:
  profiles: ["consent"]
  build: .
  ports: ["127.0.0.1:8765:8765"]
  volumes: [tokens:/data]
  command: ["node", "src/consent-cli.js"]
```

`mac/scripts/consent.sh` reads the Keychain, runs
`docker compose run --rm --service-ports consent --scopes …`, and opens the printed URL.

Why this over the alternatives:

- The browser still hits `http://localhost:8765/oauth2callback`; Docker forwards it.
  `GOOGLE_REDIRECT_URI` is unchanged and still matches the registered URI byte-for-byte
  (REQ-003-14 untouched).
- One `DATABASE_URL`, one store implementation, one encryption path, valid in both processes.
  The "record written to a different store" failure mode stops existing.
- No bind mount, so no SQLite over VirtioFS. File locking across a macOS bind mount is exactly
  where REQ-001-07 (atomic upsert) and REQ-200-06 (instance guard) would rot silently.
- Same image, so the existing image checks (T-008-01, T-008-15) already cover the consent path.

Rejected: an HTTP write endpoint (new unauthenticated surface on the most sensitive write in the
system); `docker cp` (non-atomic, and the host still needs the key and a SQLite writer).

Recorded as **ADR 0008** in step 1 — this decision will be re-litigated, so it gets written down.

### B3 — The instance guard guards workers, not the store

The consent container is not a worker and does not take the guard. SPEC-200's *Definitions*
says so explicitly in step 1.

### B4 — Dissolved by B2

`TOKEN_ENC_KEY` reaches the consent container through the identical Keychain seam as
`scripts/up.sh`. One seam, one test (T-200-16), one thing to document.

### B5 — Health binds `0.0.0.0`, Compose publishes `127.0.0.1`

Binding `127.0.0.1` **inside** a container makes the endpoint unreachable from the host, so
REQ-200-10 as literally written is unimplementable. Bind `0.0.0.0` in-container and publish
`127.0.0.1:8080:8080`. Step 1 rewords REQ-200-10 to state the observable property — *reachable
only from the host loopback interface* — rather than the mechanism, and adds `HEALTH_BIND_ADDR`
(default `0.0.0.0`) so a non-container run can still bind loopback. T-200-11 stays one test for
one requirement, at integration level over the `compose-mac` fixture, with two probes: loopback
serves the fields, the LAN interface refuses.

### B6 — T-008-11 is reinstated for `mac/`

REQ-008-11 (fail closed when the secret backend is unavailable) applies to `mac/` — the spec's
partial note excludes only REQ-008-09/10/13/14 as Vault-specific — but the test plan drops
T-008-11 as "Vault-specific". That is a MUST with no test, violating `specs/README.md` rule 1.
Step 1 moves it into inherited coverage: level unit, fixture `store-unreachable`, with the note
that the secret backend here is the encrypted store plus the injected key, and a sealed Vault is
`homelab/`'s instance of the same requirement.

### B7 — `.env.example` lists `TOKEN_ENC_KEY` by name, unset

REQ-008-07 requires every variable to be **listed**, not **set**. T-008-07 asserts exactly that.
The current file conflates the two by omitting the key entirely, so T-008-07 would fail. Step 2
adds the name with an empty value and a loud comment that it is injected at `compose up` and must
not be filled in here.

### B8 — Every open config value gets a default

None of these exist today; several are required by a MUST. Step 2 adds them to `mac/lib/config.js`,
`mac/.env.example` and the `mac/docs/deploy.md` table.

| Variable | Default | Requirement |
|---|---|---|
| `TOKEN_ENC_KEY` | *(listed, unset — injected at `compose up`)* | REQ-008-07 / B7 |
| `OAUTH_PUBLISHING_STATUS` | `testing` | REQ-002-13, REQ-007-08 |
| `POLL_INTERVAL_MS` | `300000` | REQ-006-11 |
| `POLL_INTERVAL_FLOOR_MS` | `60000`, pending derivation | REQ-006-13 |
| `DEAD_ALERT_INTERVAL_MS` | `3600000` | REQ-200-13 |
| `TICK_JITTER_PCT` | `10` | REQ-200-16, REQ-002-12 |
| `INSTANCE_GUARD_TTL_MS` | `900000` (3 ticks) | REQ-200-06, T-200-07 |
| `REFRESH_ABSENCE_WINDOW_MS` | `10800000` (3 h) | REQ-007-07 |
| `HTTP_TIMEOUT_MS` | `10000` | REQ-005-09 |
| `RETRY_BASE_MS` / `RETRY_CAP_MS` / `RETRY_MAX_ATTEMPTS` / multiplier | `500` / `30000` / `5` / `2`, full jitter | REQ-009-04, REQ-009-05 |
| `HEALTH_BIND_ADDR` | `0.0.0.0` | REQ-200-10 / B5 |
| `DRIVE_FULL_SCOPE_ENABLED` | `false` | REQ-004-09 |

**Open question, not papered over:** `REFRESH_ABSENCE_WINDOW_MS` will false-alarm every time the
lid closes overnight. Intended resolution — suppress the absence alert when the last tick observed
a resume event. Logged against REQ-007-07; it may need a superseding requirement. Closed in step 4.

### B9 — One registry shape

SPEC-004's `REGISTRY` maps operation id → scope array; SPEC-005 REQ-005-05 requires an
`OperationDescriptor` carrying `mutating` and a mandatory `idempotent`. Step 1 merges them in
SPEC-004's **interface sketch only**: `REGISTRY[id]` becomes
`{id, scopes, mutating, idempotent, idempotencyNote?}`, `checkCoverage` / `assertCoverage` /
`scopesToEscalate` read `.scopes`, and `validateRegistry` enforces REQ-004-07, REQ-004-08 and
REQ-005-05. No requirement text changes, so no ADR is needed.

### B10 — Build all four adapters; do not take the exclusion

SPEC-005 is fully in scope for `mac/` (15 requirements) but nothing in the operational path calls
the adapters. Build them anyway, in step 6.

Every existing exclusion — T-009-08 and T-005-16 (no queue), T-006-01…10 (no ingress),
T-008-09/10/13/14 (no Vault) — is *structurally impossible* in this option. "Drive has no caller
yet" is merely *unused*, a categorically weaker reason, and accepting it establishes that anything
inconvenient can be excluded, which ends the coverage gate as a meaningful artifact. Gmail is not
optional regardless: REQ-006-11 polling calls `users.history.list`, which per REQ-005-10 and
REQ-004-08 must be a registered adapter operation. Once the registry, taxonomy, token gate and
faked-HTTP harness exist, Drive/Docs/Sheets are mechanical.

Interim honesty: SPEC-005 is knowingly partial through steps 3–5 and `/option-status` reports the
drift openly. `accepted` does not mean `implemented`; the status field is the tracking device.
Only T-005-16 stays permanently excluded.

### B11 — Human-only Google Cloud work gates step 3

Cannot be automated, has real latency, and blocks step 3's exit criteria. Runs in parallel with
step 2's scaffolding. Full checklist in step 2.

### B12 — CI is red on a clean tree today

```console
$ git ls-files | grep -E '\.env$|\.env\.|\.token$|credentials\.json|client_secret'
homelab/.env.example
mac/.env.example
simple/.env.example
```

The `\.env\.` branch matches `.env.example`, which all three options track intentionally. The
`docs` job in `.github/workflows/ci.yml` has never run because there is no remote, so nobody has
seen it — the moment a remote is added, CI fails on an unmodified tree. Step 2 pipes the result
through `grep -v '\.env\.example$'`.

## The six steps

### Step 1 — Accept the mac specs, amending them so they describe something buildable

**Goal.** Move 10 specs from `draft` to `accepted`, fixing every contradiction in the same change.
`accepted` is a ratchet: afterwards each of these costs an ADR. Fixing them now is free. **Zero
code in this step.**

**Deliverables.**

| File | Change |
|---|---|
| `specs/spec-001` … `spec-009` (9 files) | `status: draft` → `accepted` |
| `mac/specs/spec-200-expiry-aware-worker.md` | `status: accepted`; REQ-200-10 reworded per B5; *Definitions* records B3; *Out of scope* consent line corrected per B2 |
| `specs/spec-004-scope-registry.md` | Interface sketch merged per B9 |
| `mac/specs/test-plan.md` | T-008-11 moved out of *Not tested here* into inherited coverage (B6); T-003-11…15 note corrected for container consent; fixtures `guard-stale` TTL and `consent-container` added |
| `mac/docs/architecture.md`, `mac/docs/deploy.md`, `mac/CLAUDE.md` | Consent-in-container correction; health publishing; drop the now-impossible troubleshooting row |
| `docs/adr/0008-consent-as-a-one-shot-container.md` | New — B2 and its rejected alternatives |

`spec-100` and `spec-300`…`303` stay `draft`. Not this option's scope.

**Exit criteria.**

- `grep -l 'status: draft' specs/spec-*.md mac/specs/spec-*.md` returns nothing.
- CI's "every MUST has a test id" check passes locally over the 10 accepted specs.
- No mac-applicable MUST appears in mac's *Not tested here*.
- `npx markdownlint-cli '**/*.md' --ignore node_modules` clean; the link check passes for the ADR.

**Resolves.** B3, B6, B9, and the spec half of B5. Records B2 and B4.

### Step 2 — Scaffold, decide every config value, make CI green, clear the human gate

**Goal.** Everything a first failing test needs, every number the specs left blank, and the
Console work. No production logic.

**Scaffolding.**

- `mac/package.json` — private, CommonJS, `engines.node: ">=20"`.
  - Runtime deps: **`google-auth-library` and `better-sqlite3` only.** Use the former *solely* for
    `OAuth2Client#verifyIdToken` (REQ-003-16 — never hand-roll JWKS verification). Do the token
    exchange and every API call with global `fetch`, not `googleapis`: the endpoint must be
    injectable so tests can fake at the HTTP boundary with a real `node:http` server and real
    `Retry-After` headers, `AbortSignal.timeout` gives REQ-005-09 for free, and the raw error body
    is needed for REQ-009-01 classification.
  - `better-sqlite3` over `node:sqlite` — synchronous, real transactions (REQ-001-07), not
    experimental. Base image `node:22-bookworm-slim` so glibc prebuilds land and no compiler
    enters the image.
  - devDeps: `c8`, `eslint@9`, `markdownlint-cli`.
- `mac/Dockerfile` — two stages, `npm ci --omit=dev`, copy `lib/ src/ package.json` only,
  `USER node`, `CMD ["node","src/index.js"]`. No `ARG`, no secret `COPY` (REQ-008-01).
- `mac/.dockerignore` — `.env`, `.git`, `tests/`, `node_modules`, `scripts/`. Load-bearing for
  T-008-01 and T-008-15.
- `mac/docker-compose.yml` — `gmail-worker` (`restart: always`, `env_file: .env`,
  `TOKEN_ENC_KEY` pass-through, `ports: ["127.0.0.1:8080:8080"]`, `volumes: [tokens:/data]`,
  `stop_grace_period: 30s` for REQ-200-08, `json-file` logging with size limits for REQ-007-13),
  plus the `consent` profile service from B2, plus `volumes: { tokens: }`.
- `mac/eslint.config.js` — flat config, CommonJS. Three rules matter more than the rest combined:
  1. Ban `Date.now` and bare `new Date()` outside an allowlist. This is the enforcement arm of
     REQ-002-03 and REQ-200-05, and lint proves it far better than T-200-05 can.
  2. Ban `process.env` outside `lib/config.js` — REQ-008-03, T-008-03.
  3. Ban `console` outside `src/*-cli.js` and `scripts/` — REQ-007-01's blast radius.
- `mac/.c8rc.json` — thresholds start at whatever step 3 actually measures, recorded here, and
  ratchet: step 4 to 90 lines / 85 branches / 90 functions; step 6 to 95/90 plus per-file 100%
  branch coverage on `refresh-engine.js`, `errors.js` and `scope-registry.js`. Do not set 90 on
  day one and then be tempted to lower it.

**Config.** Add every row of the B8 table to `mac/lib/config.js`, `mac/.env.example` and
`mac/docs/deploy.md`.

**CI and tooling.**

- `.github/workflows/ci.yml` — fix the B12 false positive; give the `code` job
  `defaults.run.working-directory: mac` and gate `npm ci` on `mac/package-lock.json`; add the c8
  step; add the Docker-gated integration job the file's own TODO already describes.
- `CLAUDE.md`, `README.md`, `.claude/skills/tdd-workflow/SKILL.md`,
  `.claude/agents/test-author.md` — correct the stale
  `src/refresh-engine.js → tests/refresh-engine.test.js` mirror claim to the B1 layout.

**Human gate — STOP, a person is required (B11).** Create the GCP project; enable the Gmail,
Drive, Docs and Sheets APIs (**not** Pub/Sub — that is `homelab/`); configure the consent screen;
add the account as a Test user; create a **Desktop app** client with
`http://localhost:8765/oauth2callback`; and fill in the quota table at
`../docs/google-cloud-setup.md` §6, deriving `POLL_INTERVAL_FLOOR_MS` from it (REQ-006-13). Also
add a git remote, because CI has never run.

**Exit criteria.**

- `cd mac && npm ci && npx eslint . && node --test` — installs clean, lints clean, reports zero
  tests honestly rather than vacuously.
- `docker compose build` succeeds; `docker history` shows no secret; the image contains no `.env`
  and no `tests/`.
- `docker compose up -d` starts and exits non-zero naming the missing variable — REQ-200-07 by
  construction, before any code proves it.
- `git ls-files | grep -E '\.env$|\.env\.' | grep -v '\.env\.example$'` is empty; CI green on the
  first push.
- No cell in `../docs/google-cloud-setup.md` §6 still reads *record it*; §7 fully ticked.
- Every variable in `lib/config.js`'s required list appears in `.env.example` — T-008-07 passing
  before it is written.

**Resolves.** B1, B4, B7, B8, B11, B12, and the code half of B5.

### Step 3 — The walking skeleton: a real token refreshing on this Mac

**This is the "it works" milestone.** One thin vertical slice, TDD'd on the critical path only,
that consents against real Google, stores an encrypted refresh token on the named volume, and
performs a real refresh — observed, not inferred.

**Deliverables.** In `mac/lib/`: `errors.js` (CLASS map, typed errors, `classify()`), `config.js`,
`secret.js`, `secrets/envelope.js`, `secrets/index.js`, `token-store.js` + `stores/sqlite.js`
(WAL, `busy_timeout`, transactional upsert), `oauth-client.js` (fetch-based exchange with an
injectable base URL — an implementation detail under SPEC-002's `deps.oauth`), `refresh-engine.js`,
`consent.js`, `logger.js`, `health.js`. In `mac/src/`: `worker.js`, `instance-guard.js`,
`index.js`, `consent-cli.js`. In `mac/scripts/`: `up.sh`, `consent.sh`.

**Scope discipline.** `consent.js` implements PKCE, state, `access_type=offline`,
`prompt=consent`, no-refresh-token detection and ID-token verification — but **not** incremental
consent (step 4). `refresh-engine.js` implements `needsRefresh`, single-flight, exchange, persist
and `invalid_grant` → DEAD — but **not** the full retry matrix (step 4). Scope checking is exact
set containment; the *registry* arrives in step 5.

**Test IDs (~30 of 138).** T-002-01…05 (the `needsRefresh` boundary matrix — the highest-value
tests in the option), T-002-06, T-002-10, T-002-19; T-001-01…03, T-001-07, T-001-09; T-003-01…03,
T-003-05, T-003-07, T-003-16; T-008-04, T-008-05; T-009-01, T-009-02; T-200-01, T-200-02,
T-200-04, T-200-08, T-200-16; T-007-01, T-007-04.

**Exit criteria — demonstrated, not asserted.**

1. `cd mac && node --test` green; `npx eslint .` clean; c8 baseline recorded here.
2. `./scripts/consent.sh --scopes gmail.readonly` opens a browser, consent completes against real
   Google, and the helper prints the account and the granted scope set. Check both — a narrower
   granted set means a scope was declined.
3. `docker run --rm -v mac_tokens:/data alpine strings /data/tokens.db | grep -c '1//'` returns
   `0`. The refresh token is not on disk in plaintext.
4. **The milestone.** With `./scripts/up.sh` running, `curl -s localhost:8080/health | jq` shows
   `state: VALID` and a `secondsUntilExpiry` that decreases across polls and then jumps back up —
   and `docker compose logs gmail-worker` shows exactly one `refreshed=true` line at that moment,
   with `refreshed=false` on every other tick. That is a real Google refresh, observed. Anything
   less is a claim.
5. `curl --max-time 3 http://<lan-ip>:8080/health` fails to connect.

**Resolves.** B2 and B4 in running code.

If this step needs splitting during execution, split it at the **consent** boundary — store,
engine and worker against a hand-seeded record first, then consent. Do **not** split it at the
Docker boundary: running outside the container hides precisely the volume and networking problems
the skeleton exists to flush out.

### Step 4 — Backfill the critical-path specs to full coverage

**Goal.** Every MUST in SPEC-001, 002, 003, 007, 008 and 009 has a passing test. Where the
skeleton stops being a demo.

**Deliverables.** Retry and backoff with full jitter and `Retry-After` honoured over computed
backoff; `UNAUTHENTICATED` refresh-once-retry-once; the circuit breaker; incremental consent with
scope merge; `AccountMismatchError`; DEAD-clearing re-consent; the day-6 age warning; `purge` with
revoke-before-delete; two-key rotation; corrupt-record isolation; `listExpiringBefore`; the full
health report with the liveness/readiness split; alerting (invalid_grant, absence,
backend-unavailable — one per backend, not one per user); correlation IDs; the redacting logger
wrapper; the Postgres store variant for T-001-14. First integration-level tests: `store-sqlite`,
`store-postgres`, `built-image-mac`, `env-incomplete`.

**Test IDs.** T-001-04…06, 08, 10…18; T-002-07…09, 11…18; T-003-04, 06, 08…15; T-007-01…14;
T-008-01…08, 11, 12, 15; T-009-01…07, 09…13. Roughly 95 cumulative of 138.

**Exit criteria.**

- `/option-status --option mac` reports zero untested MUSTs for SPEC-001, 002, 003, 007, 008, 009.
- `node --test --test-name-pattern integration` green with Docker running.
- `c8 --check-coverage` passes at 90/85/90, and the thresholds are committed at that level.
- T-008-15 **fails** a deliberately poisoned build — prove the check works by breaking it.
- T-008-07 passes against the real `mac/.env.example`.
- The B8 sleep-suppression open question is closed: implemented, or recorded as a superseding
  requirement with an ADR.

### Step 5 — Complete SPEC-200 and add the polling loop

**Goal.** The worker satisfies every one of its own requirements, and the operational loop does
something beyond refreshing.

**Deliverables.** Stale-holder TTL reclaim in `instance-guard.js`; SIGTERM mid-refresh; tick
jitter; UTC-internal with localised output only for humans; DEAD-alert throttling; the idle-tick
I/O budget (exactly one store read, one log line, zero HTTP); `lib/history-sync.js`
(`users.history.list`, cursor advance, `410` handling); push/poll mutual-exclusion config
validation; polling-floor startup rejection using the quota figure derived in step 2;
`lib/scope-registry.js` core with the merged descriptor shape; and `lib/adapters/gmail.js` read
operations as the polling loop's only legal route to Google.

**Test IDs.** T-200-03, 05…07, 09…15, 17…19; T-006-11…13; T-004-01…03, 07, 08; a subset of
T-005-01…03, 07…10. Roughly 120 cumulative.

**Exit criteria.**

- All 19 T-200 IDs pass; `/option-status --option mac` shows SPEC-200 with zero untested MUSTs.
- T-200-11 passes over the real `compose-mac` project: loopback serves the fields, the LAN
  interface refuses.
- `docker compose kill -s SIGTERM gmail-worker` during a slowed refresh exits 0, leaves a
  self-consistent record and releases the guard; a subsequent `up` starts without
  `InstanceHeldError`.
- A second worker against the same volume exits non-zero and leaves the record byte-identical.
- `POLL_INTERVAL_MS` below the floor fails startup, naming both values.
- `grep -rn 'Date.now\|new Date()' mac/lib mac/src` returns only allowlisted sites, and eslint
  enforces it.

### Step 6 — Adapters, zero drift, honest status

**Goal.** Full coverage of the mac surface, and spec statuses that finally tell the truth.

**Deliverables.** All four adapters complete — `dryRun` on every mutating operation,
`confirmPermanent` on destructives, complete pagination, per-operation timeouts, content never
logged or serialised into an error, explicit `idempotent` declarations enforced at startup. The
SPEC-004 remainder: `scopesToEscalate`, `DRIVE_FULL_SCOPE_ENABLED` gating, and
`403 insufficient_permission` logged as a registry defect. `mac/docs/e2e-checklist.md` — the
manual real-Google, real-browser procedure CI must never run. Statuses flipped to `implemented`
on every spec whose MUSTs are fully proven in `mac/`.

**Test IDs.** T-004-04…06, 09…11; T-005-01…15; the remaining T-009. **138 of 138 accounted for**,
with only the structurally impossible exclusions named in *Not tested here*.

**Exit criteria.**

- `/option-status --option mac` reports **zero drift** — no untested MUSTs, no missing rows, no
  orphan tests, no unimplemented plans, no status lies.
- `c8 --check-coverage` at 95/90 globally, plus per-file 100% branch coverage on
  `refresh-engine.js`, `errors.js` and `scope-registry.js`.
- Adding an adapter operation without a registry entry, or with `idempotent` omitted, fails
  startup — demonstrated by two deliberately broken branches.
- `/verify --option mac` returns a full-pass table.
- Every spec marked `implemented` names, here, which option proved it. A spec whose MUSTs are only
  partly provable in `mac/` — SPEC-006, SPEC-008, SPEC-009 — stays `accepted`, not `implemented`.
  Do not let the last step tell the first lie.

## Invariants that never relax

- **The clock is injected.** No inline `Date.now()` in decision code. Every interesting test in
  this option is a time test, and eslint enforces this from step 2.
- **No test sleeps.** A `setTimeout` in a test is a defect. Advance the injected clock instead.
- **No automated test calls Google.** Fake at the HTTP boundary so real status codes,
  `Retry-After` headers and error bodies drive the code. Never stub the function under test.
- **No token material in logs** — including on error paths, including truncated prefixes,
  including metric labels and span attributes.
- **Fail to start** on a missing or malformed key, or an unreachable store. A worker that starts
  without a key and creates an empty store looks healthy and is useless.
- **One instance only.** Two workers on one store double-refresh.
- **Named volume, never the container filesystem.**
- **`invalid_grant` is never retried**, at any level, under any backoff.

## Deliberately out of scope

Gmail `watch()` and Pub/Sub push (no ingress — the polling fallback replaces it); multiple Google
accounts; a queue; Vault; business logic built on top of the adapters. Anything on this list is a
request for [`../homelab/`](../homelab/), not a change here.

## References

- [`CLAUDE.md`](CLAUDE.md) — the option's own rules
- [`specs/spec-200-expiry-aware-worker.md`](specs/spec-200-expiry-aware-worker.md) and
  [`specs/test-plan.md`](specs/test-plan.md)
- [`docs/architecture.md`](docs/architecture.md), [`docs/deploy.md`](docs/deploy.md)
- [`../specs/README.md`](../specs/README.md) — the traceability rules this plan is graded against
- [`../docs/google-cloud-setup.md`](../docs/google-cloud-setup.md) — the step 2 human gate
- [`../docs/token-lifecycle.md`](../docs/token-lifecycle.md) — the state machine
