# Test Plan — `mac/`

## Convention

Test IDs owned by this option ([spec-200](spec-200-expiry-aware-worker.md)) are elaborated in
full. Test IDs from shared specs appear in the inherited-coverage tables with their level,
fixture, and any option-specific note; their "what it proves" statement lives in the shared spec
and is not restated.

**Google is never called by an automated test.** The token endpoint and the API endpoints are
faked at the HTTP boundary so error classes and retry behaviour are exercised for real.
**No test sleeps.** Time comes from the injected clock.

## Specs in scope

| Spec | Applies here because | Test IDs |
|---|---|---|
| [spec-200](spec-200-expiry-aware-worker.md) | This option's own requirements | T-200-01 … T-200-19 |
| [spec-001](../../specs/spec-001-token-store.md) | Owns a SQLite/Postgres store with an encrypted refresh token | T-001-01 … T-001-18 |
| [spec-002](../../specs/spec-002-refresh-engine.md) | The refresh decision is this option's whole point | T-002-01 … T-002-19 |
| [spec-003](../../specs/spec-003-consent-flow.md) | Desktop-app client, loopback redirect, weekly re-consent | T-003-01 … T-003-16 |
| [spec-004](../../specs/spec-004-scope-registry.md) | Incremental authorization and per-call coverage | T-004-01 … T-004-11 |
| [spec-005](../../specs/spec-005-api-adapters.md) | Gmail, Drive, Docs, Sheets adapters | T-005-01 … T-005-16 |
| [spec-006](../../specs/spec-006-watch-pubsub.md) | **Polling fallback only** | T-006-11, T-006-12, T-006-13 |
| [spec-007](../../specs/spec-007-observability.md) | Logs and health report | T-007-01 … T-007-14 |
| [spec-008](../../specs/spec-008-secret-management.md) | Encrypted column, runtime-injected key | T-008-01 … T-008-08, T-008-12, T-008-15 |
| [spec-009](../../specs/spec-009-error-taxonomy.md) | Classification and retry | T-009-01 … T-009-13 |

## Levels

| Level | Means | Runs where |
|---|---|---|
| unit | Pure logic and modules with injected deps; no network, no container | `node --test`, always |
| integration | Real store on a volume, faked Google over HTTP, real container where relevant | `node --test`, tagged; needs Docker |
| e2e | Real Google, real credentials, real browser consent | Manual, documented, never in CI |

## Elaborated cases — spec-200

| Test ID | Covers | Level | Given | When | Then | Fixture |
|---|---|---|---|---|---|---|
| T-200-01 | REQ-200-01 | unit | Worker started, clock at T | Clock advanced by one interval | Exactly one evaluation ran | `clock-controlled` |
| T-200-02 | REQ-200-02 | unit | Worker not yet started | `start()` called | One evaluation ran before any interval elapsed | `clock-controlled` |
| T-200-03 | REQ-200-03 | unit | An evaluation that outlasts the interval | Two intervals of clock advance | Evaluations ran sequentially, never concurrently | `slow-tick` |
| T-200-04 | REQ-200-04 | unit | Interval 5 min, resume threshold 2 min | Clock jumped 2 h, then 30 s | The 2 h gap forced immediate evaluation; the 30 s gap did not | `clock-jump` |
| T-200-05 | REQ-200-05 | unit | Frozen injected clock | Real time passes without clock advance | No evaluation occurs | `clock-controlled` |
| T-200-06 | REQ-200-06 | integration | One worker holding the store | A second worker started | Second exits non-zero with `InstanceHeldError`; the record is untouched | `store-sqlite` |
| T-200-07 | REQ-200-06 | integration | A guard held by a dead holder | Guard TTL elapsed on the injected clock, new worker starts | The guard is reclaimed and the worker runs | `guard-stale` |
| T-200-08 | REQ-200-07 | integration | Missing `TOKEN_ENC_KEY`; then unreachable store; then unmounted volume | Start attempted in each case | Non-zero exit each time, message names the cause, and no store file is created | `env-incomplete` |
| T-200-09 | REQ-200-08 | integration | Refresh in flight | `SIGTERM` delivered | Record is self-consistent, guard released, exit 0 | `google-slow-token` |
| T-200-10 | REQ-200-09 | integration | Valid token far from expiry, worker restarted | Restart, then one tick | State preserved and no refresh performed | `token-valid`, `store-sqlite` |
| T-200-11 | REQ-200-10 | integration | Worker running | Health endpoint requested from localhost, then from another interface | Localhost serves the specified fields; the other interface cannot connect | `compose-mac` |
| T-200-12 | REQ-200-11 | unit | Token far from expiry | One tick | One log line with `refreshed=false` and seconds remaining | `token-valid` |
| T-200-13 | REQ-200-12 | unit | Empty store | Several ticks | State reported as not-consented; no error thrown, ticking continues | `store-empty` |
| T-200-14 | REQ-200-13 | unit | Record in state `DEAD` | 20 ticks | Zero token exchanges; at most one alert per alert interval | `token-dead` |
| T-200-15 | REQ-200-14 | unit | Polling mode configured | Several ticks | History polling ran; no `watch()` request was issued | `google-history` |
| T-200-16 | REQ-200-15 | integration | Key supplied only via the environment | Start | Worker starts and can decrypt; absent the key it refuses (with T-200-08) | `env-complete` |
| T-200-17 | REQ-200-16 | unit | Two workers, jitter enabled | Many simulated intervals | Their tick times differ | `clock-controlled` |
| T-200-18 | REQ-200-17 | unit | Token far from expiry | One tick | Exactly one store read, one log line, zero HTTP requests | `token-valid` |
| T-200-19 | REQ-200-18 | unit | Store written in a non-UTC local zone | Record read back and compared | Comparison is UTC-correct; only formatted output is localised | `tz-non-utc` |

## Inherited coverage

Levels and fixtures for shared test IDs as implemented here. Notes appear only where this
option's expectations differ from the shared spec's default reading.

| Test IDs | Level | Fixtures | Note for this option |
|---|---|---|---|
| T-001-01 … T-001-04 | unit | `store-sqlite`, `token-*` | Backend is SQLite by default; Postgres variant runs the same suite (T-001-14) |
| T-001-05, T-001-06 | unit | `keyring-two-keys` | Envelope encryption with `keyId`; Vault is not used in this option |
| T-001-07 | unit | `token-valid` | Redaction wrapper |
| T-001-08 | integration | `store-sqlite` | Atomic upsert under SQLite's transaction semantics |
| T-001-09, T-001-10 | unit | `google-refresh-no-rt` | The response-omits-refresh-token case |
| T-001-11 … T-001-13 | unit | `token-partial-scopes`, `token-dead` | — |
| T-001-14 | integration | `store-sqlite`, `store-postgres` | Same caller suite against both backends |
| T-001-15 … T-001-18 | unit | `token-valid`, `store-corrupt` | — |
| T-002-01 … T-002-05 | unit | `clock-controlled`, `token-*` | `needsRefresh` boundary matrix — the highest-value tests in this option |
| T-002-06, T-002-07 | unit | `google-slow-token` | Single-flight is in-process here, not Redis |
| T-002-08 … T-002-11 | unit | `token-near-expiry`, `google-invalid-grant` | — |
| T-002-12 | unit | `clock-jump` | Overlaps T-200-04 by design: engine-level vs worker-level |
| T-002-13, T-002-14 | unit | `google-500`, `token-valid` | — |
| T-002-15 | unit | `clock-controlled` | Jitter |
| T-002-16 | unit | `token-aged-6d` | Day-6 warning; Testing-mode only |
| T-002-17 … T-002-19 | unit | `google-refresh-ok`, `google-refresh-no-rt` | — |
| T-003-01 … T-003-06 | unit | `consent-desktop` | Loopback redirect on an ephemeral port |
| T-003-07, T-003-08 | unit | `google-consent-no-rt`, `token-partial-scopes` | — |
| T-003-09, T-003-10 | unit | `id-token-other-sub` | Wrong-account protection matters here: one human, several signed-in accounts |
| T-003-11 … T-003-15 | unit | `consent-desktop` | Consent runs on the host; tests exercise the module, not the browser |
| T-003-16 | unit | `id-token-forged` | ID token verification gate. `user_id` is the store's primary key, so this is the account-takeover boundary |
| T-004-01 … T-004-11 | unit | `token-partial-scopes` | Full registry suite |
| T-005-01 … T-005-16 | unit | `google-api-*` | Gmail, Drive, Docs, Sheets adapters, all faked |
| T-006-11 | integration | `google-history` | Polling fallback |
| T-006-12 | unit | `config-both-modes` | Push is impossible here, so the exclusivity check must reject any push configuration |
| T-006-13 | unit | `google-history` | Polling floor derived from the documented quota |
| T-007-01 … T-007-14 | unit | `token-valid`, `store-unreachable` | T-007-09 uses `token-aged-6d`; T-007-10 uses `store-unreachable` in place of a sealed Vault |
| T-008-01 … T-008-03 | integration | `built-image-mac` | No credential in image or repo |
| T-008-04 … T-008-06 | unit | `keyring-two-keys` | Envelope encryption and key rotation |
| T-008-07, T-008-08 | unit | `env-complete` | `.env.example` completeness; file permissions |
| T-008-12 | unit | `google-revoke` | Revoke before purge |
| T-008-15 | integration | `built-image-mac` | Poisoned-build check |
| T-009-01 … T-009-13 | unit | `google-*` error set | Full taxonomy; T-009-08 is not applicable (no queue) and is recorded below |

## Fixtures

| Name | Shape |
|---|---|
| `clock-controlled` | Injectable clock advanced explicitly by tests |
| `clock-jump` | Clock advanced by hours between ticks (host sleep) |
| `slow-tick` | An evaluation that takes longer than the interval |
| `store-sqlite` / `store-postgres` | Real store on a temp volume |
| `store-empty` | Store present, no records |
| `store-corrupt` | One record whose ciphertext cannot be decrypted |
| `store-unreachable` | Store path or server made unavailable |
| `guard-stale` | Instance guard held by a dead holder, past its TTL |
| `token-valid` | Expiry far beyond the skew window, full scope set |
| `token-near-expiry` | Expiry inside the skew window |
| `token-expired` | Expiry in the past, refresh token intact |
| `token-dead` | Record in state `DEAD` |
| `token-partial-scopes` | Granted set narrower than requested |
| `token-aged-6d` | `refreshTokenIssuedAt` 6 days old |
| `keyring-two-keys` | Two encryption keys loaded, records under each |
| `env-complete` / `env-incomplete` | Full and deliberately incomplete environments |
| `tz-non-utc` | Process `TZ` set to a non-UTC zone |
| `consent-desktop` | Faked authorization endpoint plus loopback callback |
| `google-refresh-ok` | Token endpoint returning a new access token and expiry |
| `google-refresh-no-rt` | Refresh response with no `refresh_token` |
| `google-consent-no-rt` | Consent exchange with no `refresh_token` |
| `google-invalid-grant` | `400 invalid_grant` |
| `google-429` / `google-429-retry-after` / `google-500` / `google-503` / `google-network` | Transient failure set |
| `google-slow-token` | Token endpoint that responds slowly, for concurrency and SIGTERM tests |
| `google-api-gmail` / `-drive` / `-docs` / `-sheets` | Faked API surfaces, including two-page list responses |
| `google-history` | `users.history.list` responses, including a `410` |
| `google-revoke` | Revocation endpoint |
| `id-token-other-sub` | ID token whose `sub` differs from the expected user |
| `id-token-forged` | ID tokens with a bad signature, wrong `iss`, wrong `aud`, and an expired `exp` |
| `config-both-modes` | Configuration enabling push and polling together |
| `built-image-mac` | Image produced by this option's build |
| `compose-mac` | Compose project for integration runs |

No fixture contains a real credential. Token values are obviously fake and exist partly so that
log-redaction tests have something to search for.

## Coverage gate

- Every MUST in spec-200 has ≥1 elaborated row.
- Every test ID from every spec in scope appears above, elaborated or inherited.
- `/option-status` reports zero drift.

## Not tested here

- **T-009-08** (class preserved through queue retries) — there is no queue in this option. It is
  covered in [`../../homelab/specs/test-plan.md`](../../homelab/specs/test-plan.md).
- **T-005-16** (non-idempotent operation on an auto-retrying queue) — same reason: no queue here.
  Covered in `homelab/`.
- **T-006-01 … T-006-10, T-006-14, T-006-15** — `watch()` and push require ingress this option
  does not have. Covered in `homelab/`.
- **T-008-09 … T-008-11, T-008-13, T-008-14** — Vault-specific; covered in `homelab/`.
- Docker Desktop's own availability, and host clock correctness — stated assumptions, not
  behaviours of this code.
- Real browser consent, which is an e2e step in [`../docs/deploy.md`](../docs/deploy.md).
