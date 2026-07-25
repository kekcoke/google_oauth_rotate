# Test Plan — `simple/`

## Convention

Test IDs owned by this option ([spec-100](spec-100-cron-refresh-worker.md)) are elaborated in
full. Test IDs from shared specs appear in the inherited-coverage table with their level and
fixture; their "what it proves" statement lives in the shared spec and is not restated here.

Google's token endpoint is never called by an automated test. The refresh endpoint is always a
local stub.

## Specs in scope

| Spec | Applies here because | Test IDs |
|---|---|---|
| [spec-100](spec-100-cron-refresh-worker.md) | This option's own requirements | T-100-01 … T-100-16 |
| [spec-007](../../specs/spec-007-observability.md) | Failure must be visible from a container with no application code | T-007-01, T-007-12 |
| [spec-008](../../specs/spec-008-secret-management.md) | No credential in the image or repo; fail fast on missing config | T-008-01, T-008-02, T-008-03, T-008-05, T-008-07, T-008-15 |
| [spec-009](../../specs/spec-009-error-taxonomy.md) | The script's response handling must match the taxonomy | T-009-01, T-009-03, T-009-04, T-009-05 |

Specs 007, 008 and 009 apply **partially** here: this option is a shell script and a crontab, with
no token store, no token exchange and no Vault, so the requirements that presuppose application-level
token logic cannot be implemented. Every excluded test id is named in "Not tested here" — an
explicit exclusion is a decision, silence would be drift.

[spec-001](../../specs/spec-001-token-store.md) does not list `simple` in its `applies_to` at all:
it describes the **application's** token store, which this option assumes exists.

## Levels

| Level | Means | Runs where |
|---|---|---|
| unit | The script invoked directly against a stub HTTP server, no container | `node --test`, always |
| integration | The built container against a stub, with the real crond schedule shortened | `node --test`, tagged; needs Docker |
| e2e | Against a real application's refresh endpoint | Manual, documented, never in CI |

## Elaborated cases

| Test ID | Covers | Level | Given | When | Then | Fixture |
|---|---|---|---|---|---|---|
| T-100-01 | REQ-100-01 | integration | The built image | Container started and left for two shortened intervals | Container is still running and `crond` is the foreground process | `compose-short-interval` |
| T-100-02 | REQ-100-02 | unit | A stub counting requests | One script invocation | Exactly one POST recorded, to `REFRESH_URL` | `stub-200` |
| T-100-03 | REQ-100-03 | unit | A stub that never responds | One invocation | Script exits non-zero within `max-time` + margin, and the lock is released | `stub-hang` |
| T-100-04 | REQ-100-04 | unit | A stub returning `500` | One invocation | Script treats it as failure (non-zero exit) | `stub-500` |
| T-100-05 | REQ-100-05 | unit | A stub that responds slowly | Two invocations started together | Exactly one POST recorded | `stub-slow` |
| T-100-06 | REQ-100-06 | unit | Lock already held | One invocation | Exits 0 and writes a skip line | `lock-held` |
| T-100-07 | REQ-100-07 | unit | A stub returning a body containing `1//0gTOKENSHAPED` | One invocation | Log contains the status code and not that string | `stub-token-body` |
| T-100-08 | REQ-100-08 | integration | Container with the log volume mounted | Container recreated after a tick | Previous log lines are still present | `compose-short-interval` |
| T-100-09 | REQ-100-09 | unit | Failure threshold of 3 | Three failing ticks, then one success | Healthcheck reports unhealthy after the third, healthy after the success | `stub-500` |
| T-100-10 | REQ-100-10 | unit | Stubs for `429`, `503`, `401`, `400` | One invocation each | `429`/`503` produce two requests; `401`/`400` produce one | `stub-429`, `stub-503`, `stub-401`, `stub-400` |
| T-100-11 | REQ-100-11 | integration | The built image | Layers and filesystem inspected | No file matching the secret patterns, no credential value | `built-image` |
| T-100-12 | REQ-100-12 | unit | The script | Executed under busybox `sh` | Runs without syntax error; no bashism | none |
| T-100-13 | REQ-100-13 | integration | The running container | Process owner inspected | Matches the documented user | `compose-short-interval` |
| T-100-14 | REQ-100-14 | unit | The Dockerfile | Parsed | Base image reference contains a digest | none |
| T-100-15 | REQ-100-15 | unit | Stub requiring a shared secret; `REFRESH_SECRET` set | One invocation | Request carries the secret and succeeds; without it the stub rejects and the tick fails | `stub-auth` |
| T-100-16 | REQ-100-16 | unit | A stub returning `200` | One invocation | Exactly one log line, containing status and duration | `stub-200` |
| T-100-17 | REQ-100-17 | unit | `../docs/architecture.md` | Parsed | The failure-modes section exists and names both unconditional refresh and the interval-lag gap | none |

## Inherited coverage

| Test ID | Level | Fixture | Note for this option |
|---|---|---|---|
| T-007-01 | unit | `stub-token-body` | Scoped to the script's log file; there is no application logger here |
| T-007-12 | unit | `stub-500` | The failure cause must be readable from the default log line |
| T-008-01 | integration | `built-image` | Same assertion as T-100-11; kept distinct because the requirements differ |
| T-008-02 | unit | none | Ignore rules cover this option's `.env` and log paths |
| T-008-03 | unit | none | `REFRESH_URL` and `REFRESH_SECRET` come only from the environment |
| T-009-01 | unit | `stub-*` set | Status→class mapping as implemented by the script |
| T-009-03 | unit | `stub-429-retry-after` | `Retry-After` is honoured by the single in-tick retry |
| T-009-04 | unit | `stub-503` | Backoff before the single retry is jittered |
| T-009-05 | unit | `stub-500` | One retry is the cap; exhaustion increments the failure counter |

## Fixtures

| Name | Shape | Used by |
|---|---|---|
| `stub-200` | Local HTTP server, `200 {"ok":true}` | T-100-02, T-100-16 |
| `stub-401` | `401` | T-100-10 |
| `stub-400` | `400` | T-100-10 |
| `stub-429` | `429` | T-100-10, T-009-01 |
| `stub-429-retry-after` | `429` with `Retry-After: 2` | T-009-03 |
| `stub-500` | `500` | T-100-04, T-100-09, T-007-12, T-009-05 |
| `stub-503` | `503` once, then `200` | T-100-10, T-009-04 |
| `stub-hang` | Accepts and never responds | T-100-03 |
| `stub-slow` | Responds after a controlled delay | T-100-05 |
| `stub-token-body` | `200` with a body containing a token-shaped string | T-100-07, T-007-01 |
| `stub-auth` | Requires a shared-secret header | T-100-15 |
| `lock-held` | Lock file held by another process | T-100-06 |
| `built-image` | The image produced by `docker compose build` | T-100-11, T-100-13, T-008-01 |
| `compose-short-interval` | Compose override with a shortened cron interval for test runs | T-100-01, T-100-08, T-100-13 |

No fixture contains a real credential. Token-shaped strings are obviously fake and used only to
prove they do not reach a log.

## Coverage gate

- Every MUST in spec-100 has ≥1 elaborated row. REQ-100-17 is a documentation requirement, and
  T-100-17 asserts structurally that [`../docs/architecture.md`](../docs/architecture.md) still
  carries the accepted-limitations section — a doc requirement is still a requirement.
- Every inherited test ID appears above with a level and a fixture.
- `/option-status` reports zero drift.

## Not tested here

Excluded test IDs, named explicitly so the exclusion is a decision rather than drift.

| Excluded | Why |
|---|---|
| **T-007-02 … T-007-11, T-007-13, T-007-14** | Presuppose application-level token logic: state-transition events, refresh outcomes, a health report, liveness/readiness, correlation ids, and a logger wrapper. This option is a shell script with a log file and a failure counter. T-007-13 (bounded logs) is the host operator's job, stated in [`../docs/deploy.md`](../docs/deploy.md). |
| **T-008-04, T-008-06** | Envelope encryption and key rotation belong to whoever owns the token store — not this option. |
| **T-008-08** | This option creates no secret file; it only reads injected configuration. |
| **T-008-09 … T-008-11, T-008-13, T-008-14** | Vault-specific. Covered in [`../../homelab/specs/test-plan.md`](../../homelab/specs/test-plan.md). |
| **T-008-12** | Revocation before purge requires owning the token store. |
| **T-009-02** | `invalid_grant` comes from the token endpoint, which this option never calls. Its `401` handling is covered by T-100-10. |
| **T-009-06 … T-009-13** | Refresh-on-401, typed errors, circuit breaking, queue retry and metric labels all require application code. T-009-08 needs a queue; covered in `homelab/`. |
| **All of spec-001 … spec-006** | The application's token store, refresh exchange, consent flow, scope registry, adapters and push handling. This option implements none of them, and spec-001's `applies_to` no longer lists `simple`. |

Also out of scope:

- Whether a refresh was actually *needed*. By design this option cannot know.
- `crond`'s own scheduling accuracy.
