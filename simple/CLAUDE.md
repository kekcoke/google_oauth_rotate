# Option 1 — `simple/`

Docker + cron refresh worker. The smallest thing that can keep a token alive when an
application with its own `/oauth/refresh` endpoint already exists.

## What this option is

An Alpine container running `crond`. Every 30 minutes a shell script POSTs to the app's refresh
endpoint. The worker holds no tokens, makes no decisions, and knows nothing about OAuth — it is
a scheduler with a `curl` in it. The **application** owns the token store, the consent flow, and
the refresh logic.

## What this option is deliberately not

- **Not expiry-aware.** It refreshes whether or not the token needs it. That is the accepted
  tradeoff, recorded in [ADR 0002](../docs/adr/0002-expiry-aware-refresh-over-fixed-cron.md).
- **Not the owner of the consent flow.** Re-consent happens in the app.
- **Not multi-user aware.** One endpoint, one poke.

If you find yourself adding expiry logic here, you want [`../mac/`](../mac/) instead. Do not
grow this option into option 2 — that is what option 2 is for.

## Specs

| Spec | Role here |
|---|---|
| [spec-100](specs/spec-100-cron-refresh-worker.md) | This option's own requirements |
| [spec-001](../specs/spec-001-token-store.md) | Applies to the **app's** store, which this option assumes exists |
| [spec-007](../specs/spec-007-observability.md) | Logging and failure visibility, scoped to what a cron container can do |
| [spec-008](../specs/spec-008-secret-management.md) | No credentials in the image, ever |
| [spec-009](../specs/spec-009-error-taxonomy.md) | The script's interpretation of HTTP responses |

Test plan: [`specs/test-plan.md`](specs/test-plan.md).

## Stack

- `alpine:latest` (pin a digest in the real Dockerfile), `busybox crond`, `curl`
- Shell (`/bin/sh`, POSIX — not bash)
- Tests: `bats`-style shell assertions driven from `node:test` so one runner covers the repo, or
  plain shell asserts invoked by `node --test`. Decide in the first `/tdd-red` pass and record it.

## Local rules

- The script is POSIX `sh`, not bash. Alpine has no bash by default and adding it to run a
  10-line script is not worth the image.
- `curl` always carries `--max-time` and `--fail`. A refresh call that hangs forever holds the
  lock and blocks the next tick.
- Use `flock` (busybox) to prevent overlapping invocations.
- **Never log the response body.** A refresh endpoint that returns token material would leak it
  into the log file. Log status code and duration only.
- The log path is on a mounted volume, not the container filesystem, or it vanishes on restart.
- No credentials in the image: no `COPY` of an env file, no `--build-arg`.

## Verify a change

```sh
docker compose build
docker compose up -d
docker compose logs -f token-worker      # expect one attempt per interval
```

For tests, drive the script against a local stub endpoint that can return `200`, `401`, `429`,
`500`, and a hang. Never point tests at the real Google or at a production app.
