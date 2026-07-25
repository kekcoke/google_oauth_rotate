# `simple/` — Deploy

## Prerequisites

- Docker with Compose v2 on any host
- An application already exposing `POST /oauth/refresh`, reachable from the worker container
- A volume path for logs, and log rotation configured on the host

## Configuration

All configuration is runtime-injected. Nothing below belongs in the image or in git.

| Variable | Required | Meaning |
|---|---|---|
| `REFRESH_URL` | yes | Full URL of the application's refresh endpoint |
| `REFRESH_SECRET` | recommended | Shared secret sent with the request so anything on the network cannot trigger refreshes |
| `LOG_DIR` | yes | Log directory inside the container; must be a mounted volume |
| `FAIL_THRESHOLD` | no (default 3) | Consecutive failures before the healthcheck reports unhealthy |
| `REFRESH_INTERVAL` | no (default 30m) | Cron interval; changing it requires regenerating the crontab |

Copy `.env.example` to `.env`, fill it in, and `chmod 600 .env`. `.env` is git-ignored — keep it
that way.

## Compose shape

```yaml
services:
  token-worker:
    build: .
    restart: always
    env_file: .env
    volumes:
      - ./logs:/logs
    healthcheck:
      test: ["CMD", "/scripts/healthcheck.sh"]
      interval: 60s
      retries: 3
    depends_on:
      - app          # only if the app is in the same Compose project
```

`restart: always` matters. A worker that exits and stays down produces silence, which is the
failure mode this option is worst at detecting.

## Deploy

```sh
cp .env.example .env      # then edit, then chmod 600 .env
docker compose build
docker compose up -d
docker compose logs -f token-worker
```

Confirm within one interval that a tick has run and produced a single log line with a `2xx`.

## Verify

```sh
# a tick has run and succeeded
tail -n 5 ./logs/oauth-refresh.log

# the container is healthy
docker inspect --format '{{.State.Health.Status}}' <container>

# no credential in the image
docker history --no-trunc <image> | grep -i -E 'secret|token|client_id'   # expect nothing
```

Then verify the thing that actually matters: that the **application** now holds a token with a
future expiry. The worker succeeding only proves the endpoint returned `2xx`.

## Operational responsibilities

These are the operator's, not the container's, and this option provides no help with them:

- **Log rotation.** The log volume grows without bound otherwise. Configure `logrotate` on the
  host or Docker's `json-file` limits for the container's own stdout.
- **Alerting.** The container exposes a health status and a log file. Something must watch them.
  Absence of log lines is as important as failures in them.
- **Re-consent.** When the grant dies (weekly, in Testing mode), the application needs a new
  refresh token. See [`../../docs/runbooks/re-consent.md`](../../docs/runbooks/re-consent.md).
  The worker will keep failing identically until that happens — which is the intended signal.
- **Clock.** The container inherits the host clock; a badly wrong host clock shifts the schedule.

## Upgrading

```sh
docker compose build --pull
docker compose up -d
```

A restart may skip at most one tick. Because refreshes here are unconditional and frequent
relative to token lifetime, a single missed tick is harmless — which is the one genuine advantage
of the unconditional design.

## Rollback

Redeploy the previous image tag. The worker is stateless apart from its log volume and failure
counter; the counter resets on the next success and nothing else is lost.

## When to stop using this option

If you are here because refreshes are failing at the wrong moment, or because you cannot tell
whether a refresh was needed, the answer is not a shorter interval. It is
[`../../mac/`](../../mac/) or [`../../homelab/`](../../homelab/).
