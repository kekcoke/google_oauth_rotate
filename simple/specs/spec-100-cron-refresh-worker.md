---
id: SPEC-100
title: Cron refresh worker
status: draft
applies_to: [simple]
depends_on: [SPEC-007, SPEC-008, SPEC-009]
---

# SPEC-100 — Cron refresh worker

## Purpose

A container that wakes on a fixed schedule and asks an existing application to refresh its
OAuth tokens. It holds no credentials and makes no decisions. Its requirements are almost
entirely about not being harmful: not hanging, not overlapping, not leaking, and not failing
silently.

## Definitions

- **Refresh endpoint** — the application's `POST /oauth/refresh`, which performs the actual
  token exchange.
- **Tick** — one scheduled invocation of the script. Default interval **30 minutes**.
- **Failure counter** — a file on the log volume tracking consecutive failed ticks, read by the
  container healthcheck.

## Requirements

| ID | Level | Requirement |
|---|---|---|
| REQ-100-01 | MUST | Run `crond` in the foreground as PID 1's workload so the container stays alive and its exit is visible to Docker. |
| REQ-100-02 | MUST | Invoke the refresh endpoint once per tick, at the configured interval, with the URL supplied by the environment. |
| REQ-100-03 | MUST | Bound every request with a connect and total timeout; an unbounded request MUST NOT be possible. |
| REQ-100-04 | MUST | Treat a non-2xx response as a failure — `curl`'s default exit status ignores HTTP errors, so `--fail` (or an explicit status check) is required. |
| REQ-100-05 | MUST | Serialise invocations with a lock, so a slow tick cannot overlap the next one. |
| REQ-100-06 | MUST | Exit 0 without calling the endpoint when the lock is held, and record that the tick was skipped. |
| REQ-100-07 | MUST NOT | Write the response body to the log; log method, status, duration and a timestamp only. |
| REQ-100-08 | MUST | Write logs to a mounted volume path, not to the container's own filesystem. |
| REQ-100-09 | MUST | Maintain a consecutive-failure counter, reset on success, and expose it through a container healthcheck that reports unhealthy at the configured threshold. |
| REQ-100-10 | MUST | Retry at most once within a tick, and only for transient classes per [`spec-009`](../../specs/spec-009-error-taxonomy.md); a `4xx` other than `429` MUST NOT be retried. |
| REQ-100-11 | MUST NOT | Contain any credential in the image; the endpoint URL and any shared secret arrive by runtime injection. |
| REQ-100-12 | MUST | Be POSIX `sh` compatible — no bashisms. |
| REQ-100-13 | MUST | Run as a non-root user, or document precisely why `crond` requires root in this image. |
| REQ-100-14 | MUST | Pin the base image by digest, not by `latest`. |
| REQ-100-15 | SHOULD | Authenticate to the refresh endpoint with an injected shared secret, so anything on the network cannot trigger refreshes. |
| REQ-100-16 | SHOULD | Emit one line per tick even on success, so absence of lines is itself detectable. |
| REQ-100-17 | MUST | Document the accepted limitation that refreshes are unconditional and a 30-minute grid can lag an expiry. |

## Interface sketch

```sh
# scripts/token-refresh.sh — POSIX sh
# Env: REFRESH_URL (required), REFRESH_SECRET (optional), LOG_DIR, FAIL_THRESHOLD
# Exit: 0 tick handled (including skipped), 1 tick failed
#
#   flock -n "$LOCK" || { log skipped; exit 0; }         # REQ-100-05, REQ-100-06
#   curl --fail --silent --show-error \
#        --connect-timeout 5 --max-time 20 \             # REQ-100-03
#        -X POST "$REFRESH_URL" -o /dev/null -w '%{http_code}'
#   # one retry only for 429/5xx                          # REQ-100-10
#   # success -> reset counter; failure -> increment      # REQ-100-09
```

```text
# crontab                       REQ-100-02
*/30 * * * * /scripts/token-refresh.sh >> /logs/oauth-refresh.log 2>&1

# Dockerfile                    REQ-100-01, REQ-100-11, REQ-100-13, REQ-100-14
FROM alpine@sha256:<pinned>
RUN apk add --no-cache curl util-linux
COPY crontab /etc/crontabs/root
COPY scripts/token-refresh.sh /scripts/token-refresh.sh
HEALTHCHECK CMD /scripts/healthcheck.sh
CMD ["crond", "-f", "-l", "2"]
```

## Acceptance criteria

- [ ] `docker compose up` produces exactly one endpoint call per interval.
- [ ] A stub endpoint that hangs is abandoned at the timeout, and the next tick still runs.
- [ ] A stub returning `500` is retried once, then recorded as a failure.
- [ ] A stub returning `401` is not retried.
- [ ] Two invocations started simultaneously result in one endpoint call and one skip record.
- [ ] A stub returning a token-shaped body leaves no token material in the log.
- [ ] After the threshold of consecutive failures, `docker inspect` reports the container
      unhealthy.
- [ ] Logs survive `docker compose down && up`.
- [ ] Image inspection finds no credential.

## Tests

| Test ID | Covers | What it proves |
|---|---|---|
| T-100-01 | REQ-100-01 | The container stays running after start, and `crond` is the foreground process. |
| T-100-02 | REQ-100-02 | One tick produces exactly one POST to `REFRESH_URL`. |
| T-100-03 | REQ-100-03 | A hanging stub is abandoned at `--max-time` and the script exits non-zero. |
| T-100-04 | REQ-100-04 | A `500` response is treated as failure despite `curl` exiting 0 without `--fail`. |
| T-100-05 | REQ-100-05 | Two concurrent invocations produce one endpoint call. |
| T-100-06 | REQ-100-06 | The blocked invocation exits 0 and logs a skip. |
| T-100-07 | REQ-100-07 | A stub returning a token-shaped body leaves no such string in the log. |
| T-100-08 | REQ-100-08 | Logs are written to the mounted path and survive container recreation. |
| T-100-09 | REQ-100-09 | The counter increments on failure, resets on success, and the healthcheck flips at the threshold. |
| T-100-10 | REQ-100-10 | `429` and `503` are retried exactly once; `401` and `400` are not retried. |
| T-100-11 | REQ-100-11 | The built image contains no credential file or value. |
| T-100-12 | REQ-100-12 | The script runs under `dash`/busybox `sh` with no bashism errors. |
| T-100-13 | REQ-100-13 | The process runs as the expected non-root user, or the documented exception is present. |
| T-100-14 | REQ-100-14 | The Dockerfile references a digest-pinned base image. |
| T-100-15 | REQ-100-15 | With `REFRESH_SECRET` set, the request carries it; the stub rejects an unauthenticated call. |
| T-100-16 | REQ-100-16 | A successful tick emits exactly one log line. |
| T-100-17 | REQ-100-17 | The architecture doc contains the accepted-limitations section naming unconditional refresh and the interval-lag gap (structural check on the file). |

## Out of scope

- The refresh logic itself, the token store, and the consent flow — all owned by the application
  this worker pokes.
- Expiry-aware refresh — [`mac/specs/spec-200-expiry-aware-worker.md`](../../mac/specs/spec-200-expiry-aware-worker.md).
- Multi-user behaviour — [`homelab/`](../../homelab/).
- Alert delivery; this option produces a health signal and log lines, and the operator's
  monitoring consumes them.

## References

- [ADR 0002](../../docs/adr/0002-expiry-aware-refresh-over-fixed-cron.md)
- [`simple/docs/architecture.md`](../docs/architecture.md)
