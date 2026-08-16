# `mac/` — Deploy

Docker Desktop on macOS, one container, one named volume.

## Prerequisites

- Docker Desktop, running, set to start at login
- A Google Cloud **Desktop app** OAuth client with a loopback redirect
  ([`../../docs/google-cloud-setup.md`](../../docs/google-cloud-setup.md))
- The account added as a Test user while the app is in Testing status
- A 32-byte encryption key, base64-encoded

## Configuration

| Variable | Required | Meaning |
|---|---|---|
| `GOOGLE_CLIENT_ID` | yes | Desktop-app client id |
| `GOOGLE_CLIENT_SECRET` | yes | Desktop-app client secret |
| `GOOGLE_REDIRECT_URI` | yes | `http://localhost:<port>/oauth2callback` — must match the client exactly |
| `TOKEN_ENC_KEY` | yes | Base64 32 bytes; encrypts the refresh token at rest |
| `TOKEN_ENC_KEY_ID` | yes | Identifies the key in stored ciphertext, so it can be rotated |
| `DATABASE_URL` | yes | `sqlite:///data/tokens.db` by default, or a Postgres URL |
| `TICK_MS` | no (300000) | Evaluation interval |
| `SKEW_MS` | no (600000) | Refresh window before expiry |
| `HEALTH_PORT` | no (8080) | Health endpoint; published as `127.0.0.1:8080` and reachable from loopback only |
| `HEALTH_BIND_ADDR` | no (`0.0.0.0`) | Bind address **inside** the container. Binding `127.0.0.1` there makes the endpoint unreachable from the host, so the loopback restriction comes from the Compose port publish, not the bind (REQ-200-10). Set `127.0.0.1` only when running outside a container |

The worker refuses to start if any required variable is missing. That is deliberate — a worker
that starts without a key and creates an empty store looks healthy and is useless.

## The Keychain seam

This container **cannot read the macOS Keychain**. If the Keychain is where the key lives, the
operator's start wrapper reads it and injects it:

```sh
# scripts/up.sh — run on the host, never committed with a value in it
TOKEN_ENC_KEY="$(security find-generic-password -s google-oauth-rotate -w)" \
  docker compose up -d
```

The value passes through the environment of one command. It is not written to a file, not stored
in the image, and not committed. Note the tradeoff honestly: it is visible in that process's
environment while it runs, which is weaker than a secrets manager and the reason
[`../../homelab/`](../../homelab/) uses Vault.

## First run

```sh
cp .env.example .env && chmod 600 .env     # everything except TOKEN_ENC_KEY
docker compose build
./scripts/up.sh                            # injects the key, starts the worker
docker compose logs -f gmail-worker         # expect "not consented"
```

Then consent. The token store is a named volume, which only a container can write, so consent runs
as a **one-shot container** on the same image and the same volume — not as a host process
([ADR 0008](../../docs/adr/0008-consent-as-a-one-shot-container.md)):

```sh
./scripts/consent.sh --scopes gmail.readonly,drive.file
```

The wrapper reads the key from the Keychain and runs
`docker compose run --rm --service-ports consent`, which publishes `127.0.0.1:8765` and prints an
authorization URL. Open it; approve. The browser's callback to
`http://localhost:8765/oauth2callback` is forwarded into the container, which exchanges the code,
writes the encrypted record to the volume, and exits.

It prints the account and the granted scope set — check both. A narrower granted set than
requested means a scope was declined. The consent container takes no instance guard, so it is safe
to run while the worker is up.

Confirm the container picked it up rather than assuming it:

```sh
curl -s localhost:8080/health | jq
docker compose logs --tail 20 gmail-worker
```

## Verify

```sh
node --test                                  # unit tests
curl -s localhost:8080/health | jq           # state, seconds to expiry, token age
docker compose logs --tail 50 gmail-worker   # one decision line per tick
```

Healthy output is boring: `refreshed=false` on most ticks, one refresh roughly per token
lifetime. **Frequent refreshes are a bug**, not diligence — check `SKEW_MS` and whether
`expires_at` is being stored correctly.

Confirm the health endpoint is not reachable from elsewhere on the network:

```sh
curl --max-time 3 http://<this-mac-lan-ip>:8080/health    # expect a connection failure
```

## Ongoing operation

| Task | Cadence | How |
|---|---|---|
| Token health | daily | `/token-health --option mac` |
| Re-consent | ~weekly in Testing mode | `/reconsent --option mac` → [runbook](../../docs/runbooks/re-consent.md) |
| Key rotation | annually | [runbook](../../docs/runbooks/vault-unseal-and-rotate.md), Part 4 |
| Volume backup | weekly | Back up the named volume; losing it means re-consenting |
| Log size | — | Docker's `json-file` driver with size limits, set in Compose |

## Sleep and wake

Closing the lid pauses the container. On resume the worker detects the clock jump and
re-evaluates immediately rather than waiting for the next tick, so the first request after
waking does not fail on an expired token. If the Mac was asleep for longer than the refresh
token's life — a week, in Testing mode — expect `invalid_grant` on resume and re-consent.

## Upgrading

```sh
docker compose build --pull
./scripts/up.sh
```

State lives on the volume, so a restart loses nothing. At most one tick is skipped, and start-up
evaluates immediately.

## Troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| `InstanceHeldError` on start | A previous container still holds the guard | Confirm nothing else is running; the guard self-clears after its TTL |
| Exits immediately, non-zero | Missing required variable, or unreachable store | Read the message; it names the cause |
| `not consented` after consenting | The consent service ran without the `tokens` volume — usually `docker compose run` invoked directly, outside `consent.sh` | Re-run `./scripts/consent.sh`; confirm the `consent` service in `docker-compose.yml` still lists `volumes: [tokens:/data]` |
| Browser cannot reach `localhost:8765` | `--service-ports` omitted, so the consent container published nothing | Use `./scripts/consent.sh`, which passes it |
| `redirect_uri_mismatch` | Client's registered URI differs from `GOOGLE_REDIRECT_URI` | Make them identical, including port and path |
| `invalid_grant` | Refresh token dead — usually the 7-day Testing-mode expiry | [Re-consent](../../docs/runbooks/re-consent.md) |
| Refreshing every tick | `expires_at` not stored, or stored as a duration | Check the store; this is REQ-001-03 failing |
| Nothing in the logs at all | Docker Desktop not running | Start it; nothing inside a stopped container can report this |
