# `homelab/` — Deploy to the homelab

One Compose stack: worker, push receiver, Postgres, Redis, Vault/OpenBao. The same file deploys
to a cloud VM — see [`deploy-cloud.md`](deploy-cloud.md) for what differs.

## Prerequisites

- Docker with Compose v2, set to start on boot
- A reverse proxy terminating TLS, if Gmail push is used (Pub/Sub requires public HTTPS)
- A Google Cloud **Web application** OAuth client whose redirect URI exactly matches the
  deployment's callback URL
- A Pub/Sub topic with publish rights granted to `gmail-api-push@system.gserviceaccount.com`
- A backup target for the Postgres volume, the Vault volume, and **the Vault unseal keys**

## Assumed of the environment

This stack does not provide these, and the deployment depends on them:

- Network isolation of Postgres, Redis and Vault — none should be reachable from the LAN, let
  alone the internet
- TLS termination and a stable hostname for the push receiver
- Host-level log rotation and disk monitoring
- NTP; expiry comparisons assume a correct clock

## Services

```yaml
services:
  worker:
    build: .
    restart: always
    environment:
      DATABASE_URL: postgres://…
      REDIS_URL: redis://redis:6379
      VAULT_ADDR: http://vault:8200
      VAULT_ROLE_ID: ${VAULT_ROLE_ID}
      VAULT_SECRET_ID_FILE: /run/secrets/vault_secret_id
      IS_SCHEDULER: "true"        # exactly one replica may set this
    secrets: [vault_secret_id]
    depends_on: [postgres, redis, vault]

  receiver:                        # Pub/Sub push endpoint
    build: .
    command: ["node", "src/receiver.js"]
    restart: always
    environment:
      PUBSUB_AUDIENCE: ${PUBSUB_AUDIENCE}

  postgres:
    image: postgres:16
    restart: always
    volumes: [pgdata:/var/lib/postgresql/data]

  redis:
    image: redis:7
    restart: always
    command: ["redis-server", "--appendonly", "yes"]   # persistence, or schedules vanish
    volumes: [redisdata:/data]

  vault:
    image: hashicorp/vault:latest    # or openbao
    restart: always
    cap_add: [IPC_LOCK]
    volumes: [vaultdata:/vault/file]

volumes: { pgdata: {}, redisdata: {}, vaultdata: {} }
secrets:
  vault_secret_id: { file: ./secrets/vault_secret_id }   # git-ignored, mode 600
```

Two details worth not skipping: Redis **persistence** (an ephemeral Redis silently loses
repeatable job definitions on restart), and `IS_SCHEDULER` on **exactly one** replica (two
schedulers duplicate every scheduled job).

## First-time setup

### 1. Bring up the infrastructure

```sh
docker compose up -d postgres redis vault
```

### 2. Initialise and unseal Vault

Follow [`../../docs/runbooks/vault-unseal-and-rotate.md`](../../docs/runbooks/vault-unseal-and-rotate.md).
Back up the unseal and recovery keys to separate secure locations **before** storing any token —
losing them loses every refresh token, meaning every account must re-consent. Test the backup by
reading it.

### 3. Configure Vault

- Enable KV v2 at `secret/`, with version retention set and documented
- Enable a file audit device
- Write the AppRole policy from [spec-302](../specs/spec-302-vault-integration.md)
- Create the worker AppRole; put `role_id` in the environment and `secret_id` in
  `./secrets/vault_secret_id` (mode `600`, git-ignored)
- Store the Google client id and secret in Vault

No root token in the worker's environment, in any environment other than a throwaway test one.

### 4. Migrate the database

```sh
docker compose run --rm worker node src/migrate.js
```

### 5. Start the workers

```sh
docker compose up -d worker receiver
curl -s localhost:8080/health | jq
```

Expect `ready: true`, `secrets: "ok"`, and an empty user list.

### 6. Consent the first user

Open the deployment's `/oauth/start` in a browser and complete consent. Then confirm what
actually matters:

```sh
curl -s localhost:8080/health | jq '.users'   # state, expiry, granted scopes
```

Confirm the refresh token landed in Vault and that Postgres holds only a path:

```sh
docker compose exec postgres psql -U … -c \
  'select user_id, refresh_token_path, expires_at, granted_scopes from oauth_tokens;'
```

Any token material in that output is a bug, not a cosmetic issue.

### 7. Register Gmail push

Confirm the Pub/Sub topic grant, then register `watch()` for the user and verify a real message
produces a push. A successful `watch()` response is not proof the receiver works — see
[`../../docs/runbooks/watch-renewal.md`](../../docs/runbooks/watch-renewal.md).

## Verify

```sh
node --test                                        # unit
node --test --test-name-pattern integration        # needs Docker
curl -s localhost:8080/health | jq
docker compose logs --tail 100 worker
```

Then the deliberate failure drills, which are the only way to know the fail-closed paths work:

```sh
docker compose exec vault vault operator seal
curl -s localhost:8080/health | jq '.ready, .secrets'   # false, "sealed"
# expect exactly one alert, not one per user
docker compose exec vault vault operator unseal …       # recovers without a worker restart

docker compose stop redis
# an API call must still refresh and succeed — this is spec-301 working
docker compose start redis
```

## Ongoing operation

| Task | Cadence | How |
|---|---|---|
| Token health | daily | `/token-health --option homelab` |
| Watch renewal check | hourly (automated), reviewed daily | `/watch-renew-check` |
| Re-consent | ~weekly per user in Testing mode | `/reconsent --option homelab --user <id>` |
| Vault `secret_id` rotation | quarterly | [runbook](../../docs/runbooks/vault-unseal-and-rotate.md) Part 2 |
| Client secret rotation | annually | Same runbook, Part 3 — it forces re-consent for everyone |
| Rotation drill | monthly | `/rotate-drill` |
| Dependency and CVE sweep | weekly | `/deps-audit` |
| Backups | daily | Postgres dump, Vault volume, and verify a restore quarterly |

## Upgrading

```sh
docker compose build --pull
docker compose run --rm worker node src/migrate.js
docker compose up -d
```

Migrations must be safe against a running worker (REQ-303-12). Compose has no rolling update, so
expect a brief gap in scheduled sweeps; on-demand refresh covers the window, which is the whole
argument for [spec-301](../specs/spec-301-refresh-on-demand-guard.md).

## Rollback

1. Redeploy the previous image tag.
2. Do **not** roll the schema back — migrations are forward-only. If a migration must be undone,
   follow its documented rollback note as a new forward migration.
3. Vault KV v2 retains previous versions, so a bad secret write is recoverable.

## Troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| Every user failing at once | Vault sealed, or Postgres down | Health endpoint names which; unseal or restore |
| One user failing, others fine | That grant is dead | [Re-consent](../../docs/runbooks/re-consent.md) |
| Duplicate scheduled work | Two replicas with `IS_SCHEDULER` | Exactly one |
| Schedules vanished after restart | Redis persistence off | Enable `appendonly` |
| Pushes stopped, no errors | `watch()` lapsed | [Watch renewal](../../docs/runbooks/watch-renewal.md) |
| Pub/Sub backlog growing | Receiver unreachable or rejecting | Check TLS, the OIDC audience, and receiver logs |
| Worker won't start after deploy | Schema newer than the code | Deploy the matching version; do not force it |
| A user blocked and never refreshing | Lock held by a dead worker | It clears at the TTL; if not, that is a bug worth a test |
