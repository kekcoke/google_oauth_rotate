# Option 3 — `homelab/`

Multi-user OAuth token rotation. Docker Compose on the homelab box, portable unchanged to a
cloud VM. This is the production-shaped option.

## What this option is

The source material's Option 3 (local Docker, persistent volumes, cloud-deployment friendly)
combined with its multi-user production recommendation:

- **BullMQ on Redis** for job orchestration, per-user isolation, retry and repeatable schedules
- **Postgres** token table keyed by `user_id`
- **Refresh on demand, immediately before the API call** — the timer sweep is a safety net, not
  the mechanism
- **Per-user locking** so two concurrent jobs cannot double-refresh one account
- **Vault/OpenBao** holds refresh tokens; Postgres stores only the Vault path
- **Gmail `watch()` + Pub/Sub push**, with its own independent 7-day renewal clock

## The load-bearing idea

Correctness does not depend on the scheduler. Any code path that is about to call Google checks
expiry and refreshes first, under a per-user lock
([spec-301](specs/spec-301-refresh-on-demand-guard.md)). If Redis is down, if the sweep is
paused, if the worker just restarted — an API call still gets a valid token. The sweep exists to
keep tokens warm and to notice problems early, not to be the thing that makes the system work.

## Specs

| Spec | Role here |
|---|---|
| [spec-300](specs/spec-300-multi-user-queue-worker.md) | Queue topology, per-user locking, repeatable jobs, replicas |
| [spec-301](specs/spec-301-refresh-on-demand-guard.md) | The pre-call guard — the correctness boundary |
| [spec-302](specs/spec-302-vault-integration.md) | AppRole auth, KV paths, leases, fail-closed |
| [spec-303](specs/spec-303-schema-and-migrations.md) | Postgres schema, indexes, migrations |
| [spec-001](../specs/spec-001-token-store.md) … [spec-009](../specs/spec-009-error-taxonomy.md) | All shared specs apply; this is the only option implementing every one |

Test plan: [`specs/test-plan.md`](specs/test-plan.md).

## Stack

- Node ≥18, CommonJS, `node:test` + c8
- BullMQ + Redis; Postgres; Vault or OpenBao
- Docker Compose (see [ADR 0007](../docs/adr/0007-docker-compose-portable-deploy.md))
- `google-auth-library` / `googleapis`

## Local rules

- **Nothing is single-user.** Every function that touches a token takes a `user_id`. A module-level
  "current token" is a bug.
- **Users hold different scope sets.** Never assume a uniform grant across users
  ([spec-004](../specs/spec-004-scope-registry.md)).
- **Locks expire.** A crashed worker must not block an account permanently — every lock has a TTL
  and the TTL path is tested.
- **Fail closed on Vault.** A sealed Vault means refusing to serve tokens, never falling back to
  a cache or a plaintext copy. One infrastructure alert, not one alert per user.
- **Postgres never holds a refresh token.** Only the Vault path. A test asserts this.
- **`invalid_grant` is never retried** — not in code, and not by BullMQ's own retry mechanism.
  That second one is easy to miss.
- **Two 7-day clocks exist**: the refresh token's (Testing mode) and `watch()`'s. They are
  independent and each has its own scheduled check.
- **Push payloads are untrusted.** Verify the OIDC token, then re-fetch state from Gmail.

## Verify a change

```sh
node --test                                    # unit
node --test --test-name-pattern integration    # needs Docker: Postgres, Redis, dev Vault
docker compose up -d
curl -s localhost:8080/health | jq
```

Then `/option-status --option homelab` for requirement→test→implementation coverage.

## Deploying

[`docs/deploy-homelab.md`](docs/deploy-homelab.md) and
[`docs/deploy-cloud.md`](docs/deploy-cloud.md). The Compose file is the same in both; only secret
sourcing, TLS termination and backups differ.
