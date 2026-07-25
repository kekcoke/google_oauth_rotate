# Test Plan — `homelab/`

## Convention

Test IDs owned by this option (spec-300 … spec-303) are elaborated in full. Test IDs from shared
specs appear in the inherited-coverage table with their level, fixture, and any option-specific
note; their "what it proves" statement lives in the shared spec.

This is the only option implementing every shared spec, so it is the only test plan with complete
inherited coverage.

**Google is never called by an automated test.** **No test sleeps** — time is injected, including
lock TTLs and lease expiry.

## Specs in scope

| Spec | Test IDs |
|---|---|
| [spec-300](spec-300-multi-user-queue-worker.md) — queue worker | T-300-01 … T-300-20 |
| [spec-301](spec-301-refresh-on-demand-guard.md) — on-demand guard | T-301-01 … T-301-14 |
| [spec-302](spec-302-vault-integration.md) — Vault | T-302-01 … T-302-16 |
| [spec-303](spec-303-schema-and-migrations.md) — schema | T-303-01 … T-303-17 |
| [spec-001](../../specs/spec-001-token-store.md) — token store | T-001-01 … T-001-17 |
| [spec-002](../../specs/spec-002-refresh-engine.md) — refresh engine | T-002-01 … T-002-19 |
| [spec-003](../../specs/spec-003-consent-flow.md) — consent | T-003-01 … T-003-16 |
| [spec-004](../../specs/spec-004-scope-registry.md) — scopes | T-004-01 … T-004-11 |
| [spec-005](../../specs/spec-005-api-adapters.md) — adapters | T-005-01 … T-005-16 |
| [spec-006](../../specs/spec-006-watch-pubsub.md) — watch/push | T-006-01 … T-006-15 |
| [spec-007](../../specs/spec-007-observability.md) — observability | T-007-01 … T-007-14 |
| [spec-008](../../specs/spec-008-secret-management.md) — secrets | T-008-01 … T-008-15 |
| [spec-009](../../specs/spec-009-error-taxonomy.md) — errors | T-009-01 … T-009-13 |

## Levels

| Level | Means | Runs where |
|---|---|---|
| unit | Injected deps, no I/O | `node --test`, always |
| integration | Real Postgres, Redis and dev-mode Vault in containers; Google faked over HTTP | `node --test` tagged; needs Docker |
| e2e | Real Google, real credentials, real Pub/Sub | Manual, documented, never in CI |

Multi-worker behaviour is tested with two real worker processes against one Redis, not with
mocks. Locking bugs do not reproduce in a single process.

## Elaborated cases — spec-300 (queue worker)

| Test ID | Covers | Level | Given | When | Then | Fixture |
|---|---|---|---|---|---|---|
| T-300-01 | REQ-300-01 | unit | Job schemas | A payload without `user_id` enqueued | Rejected before processing | `queue-real` |
| T-300-02 | REQ-300-02 | integration | Two workers, one user with a due token | Both process simultaneously | Exactly one token exchange recorded | `two-workers`, `token-near-expiry` |
| T-300-03 | REQ-300-02 | integration | 50 users all due | All processed concurrently | 50 exchanges; wall time far below serialised time | `users-50` |
| T-300-04 | REQ-300-03 | integration | Lock held by a killed worker | TTL elapsed on the injected clock | Lock is acquirable | `lock-orphaned` |
| T-300-05 | REQ-300-03 | integration | Same as above | A second worker reclaims and refreshes | Refresh completes; record self-consistent | `lock-orphaned` |
| T-300-06 | REQ-300-04 | integration | A slow but live holder | Work continues past the TTL, extending; then a holder that fails to extend | Live holder is not pre-empted; failing holder raises `LockLostError` and aborts before writing | `google-slow-token` |
| T-300-07 | REQ-300-05 | integration | One user failing every attempt, plus a healthy user | Both queued repeatedly | Healthy user's latency stays under the threshold | `token-dead`, `token-near-expiry` |
| T-300-08 | REQ-300-06 | unit | Taxonomy classes | `retryOptionsFor` called for each | Transient → retries with jittered backoff; permanent → single attempt | none |
| T-300-09 | REQ-300-07 | integration | A user whose refresh returns `invalid_grant` | Job processed | BullMQ attempt count is 1; no re-enqueue | `google-invalid-grant` |
| T-300-10 | REQ-300-08 | integration | Two replicas both starting | Both register schedules | One repeatable job per schedule exists | `two-workers` |
| T-300-11 | REQ-300-09 | integration | Two replicas with jitter | Many scheduled fires observed | Fire times differ | `two-workers` |
| T-300-12 | REQ-300-10 | integration | Redis stopped | An API call made through the guard | Sweeps are absent; the call refreshes and succeeds | `redis-down` |
| T-300-13 | REQ-300-11 | integration | Redis restarted | Worker reconnects | Schedules re-established exactly once | `redis-restart` |
| T-300-14 | REQ-300-12 | integration | A job in flight | `SIGTERM` | Lock released, work completed or requeued, exit 0 | `google-slow-token` |
| T-300-15 | REQ-300-13 | integration | Jobs queued and failing | Metrics scraped | Depth, active and failure counts present per queue | `queue-real` |
| T-300-16 | REQ-300-14 | integration | A job failing past the cap | Retries exhausted | Job lands in the dead-letter destination | `google-500` |
| T-300-17 | REQ-300-15 | integration | A `DEAD` user and a healthy user | Sweep runs | `DEAD` user skipped; after re-consent it is included again | `token-dead` |
| T-300-18 | REQ-300-16 | unit | Circuit threshold 3 | Three transient failures for user A | A's circuit open, B's closed; A closes after cooldown | `clock-controlled` |
| T-300-19 | REQ-300-17 | integration | Jobs enqueued | Payloads read directly from Redis | No token material or message content | `queue-real` |
| T-300-20 | REQ-300-18 | integration | One job | Delivered twice | One net effect | `queue-real` |

## Elaborated cases — spec-301 (on-demand guard)

| Test ID | Covers | Level | Given | When | Then | Fixture |
|---|---|---|---|---|---|---|
| T-301-01 | REQ-301-01 | integration | Sweep disabled, token expired | One API call | Refresh happens and the call succeeds | `token-expired`, `sweep-off` |
| T-301-02 | REQ-301-02 | integration | Two callers, refresh interleaved between check and refresh | Both proceed | One exchange only | `two-workers`, `token-expired` |
| T-301-03 | REQ-301-03 | integration | 20 concurrent calls, one expired token | All issued together | One exchange; all 20 succeed | `token-expired` |
| T-301-04 | REQ-301-04 | integration | A holder that never completes | Waiters wait | Waiters time out with a transient error; none hangs | `google-hang-token` |
| T-301-05 | REQ-301-05 | unit | User granted `gmail.readonly`, operation needs `gmail.send` | Call attempted | Throws before any refresh and any HTTP request | `token-partial-scopes` |
| T-301-06 | REQ-301-06 | integration | Redis unavailable | API call with an expired token | Refresh succeeds; call succeeds | `redis-down` |
| T-301-07 | REQ-301-07 | unit | Refresh fails permanently | Call attempted | Call fails; the expired token is never sent to Google | `google-invalid-grant` |
| T-301-08 | REQ-301-08 | integration | `invalid_grant` on refresh | Call attempted | One exchange, record `DEAD`, one alert, call fails | `google-invalid-grant` |
| T-301-09 | REQ-301-09 | unit | Adapter module surface | Constructed and inspected | No export accepts a raw token; no Google call path bypasses the guard | none |
| T-301-10 | REQ-301-10 | unit | Token far from expiry | One call | Zero token-endpoint requests, no lock acquired | `token-valid` |
| T-301-11 | REQ-301-11 | unit | Success, thrown error, and timeout paths | Each exercised | Lock released in all three | `clock-controlled` |
| T-301-12 | REQ-301-12 | unit | One guarded call | Logs captured | Guard decision and API call share a correlation id | `token-valid` |
| T-301-13 | REQ-301-13 | integration | Another holder refreshes between fast-path check and lock acquisition | Call proceeds | Re-check inside the lock skips the redundant refresh | `two-workers`, `token-near-expiry` |
| T-301-14 | REQ-301-14 | unit | No record for the user | Call attempted | `NotConsentedError`; no record created | `store-empty` |

## Elaborated cases — spec-302 (Vault)

| Test ID | Covers | Level | Given | When | Then | Fixture |
|---|---|---|---|---|---|---|
| T-302-01 | REQ-302-01 | integration | Dev Vault | Token written then read | Present at `secret/oauth/<user>/refresh` | `vault-dev` |
| T-302-02 | REQ-302-02 | integration | A stored user | Raw Postgres columns inspected | A path, no token material | `pg-real`, `vault-dev` |
| T-302-03 | REQ-302-03 | integration | AppRole configured | Worker authenticates | Login succeeds; a root-token config is refused outside test mode | `vault-approle` |
| T-302-04 | REQ-302-04 | unit | `secret_id` from an injected file | Config loaded | Accepted; a repo-path source is refused | `vault-approle` |
| T-302-05 | REQ-302-05 | integration | The worker policy | Read attempted outside `secret/data/oauth/*` | `PermissionDeniedError` | `vault-approle` |
| T-302-06 | REQ-302-06 | integration | Lease near expiry | Renewal succeeds, then fails | Renewed first; failure → backend-unavailable | `vault-short-lease`, `clock-controlled` |
| T-302-07 | REQ-302-07 | integration | Vault sealed | Token read attempted | `BackendUnavailableError`; no cached or env value returned | `vault-sealed` |
| T-302-08 | REQ-302-08 | integration | Vault sealed | Health checked | Live true, ready false | `vault-sealed` |
| T-302-09 | REQ-302-09 | integration | Ten users, Vault sealed | All attempt operations | Exactly one alert | `vault-sealed`, `users-10` |
| T-302-10 | REQ-302-10 | integration | Sealed then unsealed | No restart | Reads resume | `vault-sealed` |
| T-302-11 | REQ-302-11 | integration | A token overwritten | Previous KV version requested | Prior value retrievable | `vault-dev` |
| T-302-12 | REQ-302-12 | integration | Auth and reads performed | Logs captured | Paths present, no secret values | `vault-dev` |
| T-302-13 | REQ-302-13 | integration | A stored user | Purged | Revocation at Google precedes Vault deletion | `google-revoke`, `vault-dev` |
| T-302-14 | REQ-302-14 | integration | Audit device enabled | A refresh-token read | Read appears in audit output with an identity | `vault-audit` |
| T-302-15 | REQ-302-15 | integration | Vault returning `503`, then a permission error | Reads attempted | `503` retried with backoff; permission error not retried | `vault-flaky` |
| T-302-16 | REQ-302-16 | integration | Client secret absent from Vault | Worker started | Start-up fails naming the missing secret | `vault-dev` |

## Elaborated cases — spec-303 (schema)

| Test ID | Covers | Level | Given | When | Then | Fixture |
|---|---|---|---|---|---|---|
| T-303-01 | REQ-303-01 | integration | Migrated database | Two users with one email, different `sub` | Two distinct rows | `pg-real` |
| T-303-02 | REQ-303-02 | integration | Migrated database | Columns introspected | No refresh-token column; `refresh_token_path` present | `pg-real` |
| T-303-03 | REQ-303-03 | integration | Session `TimeZone` set to a non-UTC zone | Instant written, read in another zone | Same instant | `pg-real`, `tz-non-utc` |
| T-303-04 | REQ-303-04 | integration | Scope array written | Read back | Exact strings preserved | `pg-real` |
| T-303-05 | REQ-303-05 | integration | Migrated database | `state = 'BANANA'` inserted | Rejected by the constraint | `pg-real` |
| T-303-06 | REQ-303-06 | integration | A full row | Written and read | All lifecycle timestamps round-trip | `pg-real` |
| T-303-07 | REQ-303-07 | integration | A user with and without a watch | Rows written | Watch fields nullable and correct | `pg-real` |
| T-303-08 | REQ-303-08 | integration | An access token stored | Raw column inspected | No plaintext; key id present | `pg-real` |
| T-303-09 | REQ-303-09 | integration | 10,000 seeded rows | `EXPLAIN` the due-token query | Index scan, not a sequential scan | `pg-seeded` |
| T-303-10 | REQ-303-10 | integration | Seeded rows including `DEAD` | Excluding query explained | No table scan | `pg-seeded` |
| T-303-11 | REQ-303-11 | integration | Fresh database | `migrate()` run twice | Second run is a no-op; migrations recorded | `pg-real` |
| T-303-12 | REQ-303-12 | integration | Worker under load | Migrations applied | No failed token operation | `pg-real`, `load-generator` |
| T-303-13 | REQ-303-13 | integration | Schema newer than the code expects | Worker started | `SchemaVersionError` | `pg-future-schema` |
| T-303-14 | REQ-303-14 | integration | An existing row | Any update | `updated_at` advances | `pg-real` |
| T-303-15 | REQ-303-15 | integration | Migrated database | Insert without `refresh_token_path` | Rejected by the database | `pg-real` |
| T-303-16 | REQ-303-16 | unit | Migration files | Parsed | Each carries a rollback note | none |
| T-303-17 | REQ-303-17 | integration | A user's email changed | Display attribute updated | Credential row untouched | `pg-real` |

## Inherited coverage

| Test IDs | Level | Fixtures | Note for this option |
|---|---|---|---|
| T-001-01 … T-001-04 | integration | `pg-real`, `token-*` | Postgres backend |
| T-001-05, T-001-06 | integration | `vault-dev` | "Encryption at rest" is satisfied by Vault; `keyId` is the KV version |
| T-001-07 | unit | `token-valid` | Redaction wrapper |
| T-001-08 | integration | `pg-real` | Atomicity across the Postgres row **and** the Vault write — the interesting case here |
| T-001-09, T-001-10 | integration | `google-refresh-no-rt` | — |
| T-001-11 … T-001-13 | integration | `token-partial-scopes`, `token-dead` | — |
| T-001-14 | integration | `pg-real`, `vault-dev` | Same caller suite as `mac/` runs against SQLite |
| T-001-15 … T-001-17 | integration | `vault-dev`, `store-corrupt` | An unreadable Vault entry must not break other users |
| T-002-01 … T-002-05 | unit | `clock-controlled` | `needsRefresh` boundary matrix |
| T-002-06, T-002-07 | integration | `two-workers` | Single-flight is the **Redis** lock here, not in-process |
| T-002-08 … T-002-11 | integration | `google-invalid-grant` | — |
| T-002-12 | unit | `clock-jump` | Less critical than on a laptop, but a paused VM behaves the same |
| T-002-13, T-002-14 | unit | `google-500` | — |
| T-002-15 | integration | `two-workers` | Jitter across replicas |
| T-002-16 | unit | `token-aged-6d` | Day-6 warning per user |
| T-002-17 … T-002-19 | integration | `google-refresh-*` | — |
| T-003-01 … T-003-16 | integration | `consent-web` | **Web-app** client on the HTTPS callback; `state` handling is load-bearing here in a way it is not for `mac/` |
| T-003-16 | unit | `id-token-forged` | ID token verification gate; with many users a forged `sub` could overwrite another account's grant |
| T-004-01 … T-004-11 | unit | `users-mixed-scopes` | Per-user scope sets differ — the multi-user case |
| T-005-01 … T-005-16 | integration | `google-api-*` | All calls go through the guard (T-301-09) |
| T-006-01 … T-006-10 | integration | `pubsub-push`, `google-history` | Full push path, including OIDC verification and replay |
| T-006-11 … T-006-13 | integration | `google-history` | Polling retained as fallback |
| T-006-14, T-006-15 | integration | `token-dead`, `pubsub-dlq` | — |
| T-007-01 … T-007-14 | integration | `vault-sealed`, `token-aged-6d` | T-007-09/T-007-10 use the sealed Vault — the canonical "one alert, not N" case |
| T-008-01 … T-008-08 | integration | `built-image-homelab` | — |
| T-008-09 … T-008-11 | integration | `vault-dev`, `vault-sealed` | Vault-specific; only this option covers them |
| T-008-12 … T-008-15 | integration | `google-revoke`, `vault-audit` | — |
| T-009-01 … T-009-07 | unit | `google-*` error set | — |
| T-009-08 | integration | `queue-real` | Class preserved through BullMQ retries — only this option can cover it |
| T-009-09 | integration | `vault-sealed`, `users-10` | — |
| T-009-10 … T-009-13 | unit | `google-*` error set | — |

## Fixtures

| Name | Shape |
|---|---|
| `clock-controlled` / `clock-jump` | Injectable clock; large jumps for paused-host cases |
| `pg-real` | Postgres container, migrated |
| `pg-seeded` | 10,000 rows for index assertions |
| `pg-future-schema` | Migrations table claiming a newer version |
| `vault-dev` | Dev-mode Vault, unsealed |
| `vault-approle` | AppRole plus the worker policy |
| `vault-sealed` | Vault sealed mid-test |
| `vault-short-lease` | Lease short enough to exercise renewal |
| `vault-flaky` | Vault returning `503`, then a permission error |
| `vault-audit` | File audit device enabled |
| `queue-real` / `redis-down` / `redis-restart` | Real Redis, and its absence and restart |
| `two-workers` | Two real worker processes against one Redis and Postgres |
| `lock-orphaned` | Lock held by a killed holder |
| `sweep-off` | Repeatable jobs disabled, to prove the guard stands alone |
| `users-10` / `users-50` / `users-mixed-scopes` | Multi-user sets, including differing grants |
| `token-valid` / `-near-expiry` / `-expired` / `-dead` / `-partial-scopes` / `-aged-6d` | Token states |
| `store-empty` / `store-corrupt` | No record; unreadable secret entry |
| `google-refresh-ok` / `-no-rt` / `-invalid-grant` / `-429` / `-429-retry-after` / `-500` / `-503` / `-network` / `-slow-token` / `-hang-token` | Faked token endpoint behaviours |
| `google-api-gmail` / `-drive` / `-docs` / `-sheets` | Faked API surfaces, with pagination |
| `google-history` | `users.history.list` responses, including `410` |
| `google-revoke` | Revocation endpoint |
| `consent-web` | Faked authorization endpoint plus the HTTPS callback |
| `id-token-forged` | ID tokens with a bad signature, wrong `iss`, wrong `aud`, and an expired `exp` |
| `pubsub-push` / `pubsub-dlq` | Push deliveries with valid, invalid and replayed OIDC tokens; dead-letter path |
| `built-image-homelab` | Image produced by this option's build |
| `load-generator` | Concurrent token operations during a migration |
| `tz-non-utc` | Non-UTC session timezone |

No fixture contains a real credential.

## Coverage gate

- Every MUST in spec-300 … spec-303 has ≥1 elaborated row.
- Every test ID from every shared spec appears in the inherited table — this option has no
  "not applicable" exclusions among them.
- `/option-status --option homelab` reports zero drift.

## Not tested here

- Vault's own sealing cryptography, Postgres's own durability, BullMQ's internals.
- Real Pub/Sub delivery, real consent in a browser, and real Google quota behaviour — all e2e,
  documented in [`../docs/deploy-homelab.md`](../docs/deploy-homelab.md).
- Kubernetes deployment — deferred in [ADR 0007](../../docs/adr/0007-docker-compose-portable-deploy.md).
- Homelab network security (firewall, VPN, reverse proxy), an assumed dependency.
