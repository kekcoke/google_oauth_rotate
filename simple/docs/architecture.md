# `simple/` — Architecture

```text
Docker Compose

+----------------+
| App Container  |
|                |
| Gmail API      |
| Uses tokens    |
+-------+--------+
        ^
        | POST /oauth/refresh   (every 30 min)
        |
+-------+--------+
| Token Worker   |
| crond -f       |
| token_refresh  |
+----------------+
        (holds no tokens)

+----------------+
| Token Storage  |
| Postgres/Redis |   owned and accessed by the App, not the worker
+----------------+
```

## Components

| Component | Responsibility | Owns tokens? |
|---|---|---|
| App container | OAuth logic, token store access, Google API calls, the `/oauth/refresh` endpoint | yes |
| Token worker | Wake on schedule, call the endpoint, record the outcome | **no** |
| Token storage | Persist tokens per [`spec-001`](../../specs/spec-001-token-store.md) | n/a |

The worker's ignorance is the point. It cannot leak a token it never holds, and it needs no
credentials beyond an optional shared secret for calling the endpoint.

## Sequence

```text
crond fires (*/30)
   |
   v
flock: another invocation running?  --yes--> exit 0, log "skipped"
   | no
   v
curl -X POST --fail --max-time 20 $REFRESH_URL
   |
   +-- 2xx ------> log success, reset failure counter
   +-- 4xx ------> log status, increment failure counter   (no retry: permanent)
   +-- 429/5xx --> one delayed retry, then log and increment
   +-- timeout --> log timeout, increment counter
   |
   v
failure counter >= threshold ? --> mark unhealthy (healthcheck reads the marker)
```

## Reference implementation shape

From the source material, with the safety requirements of
[`spec-100`](../specs/spec-100-cron-refresh-worker.md) applied:

```text
token_refresh.sh    POSIX sh: flock, curl --fail --max-time, status-only logging
crontab             */30 * * * * /scripts/token_refresh.sh >> /logs/oauth-refresh.log 2>&1
Dockerfile          FROM alpine, apk add curl, COPY crontab + script, CMD ["crond","-f","-l","2"]
compose.yaml        restart: always, log volume, REFRESH_URL from the environment
```

Note the departures from the source's sketch, each of which is a requirement rather than a
preference: log to a **mounted volume** (`/var/log` inside a container is lost on recreate),
`--fail --max-time` on `curl` (an unbounded hang blocks every later tick), `flock` (overlapping
invocations double-refresh), and status-only logging (bodies may carry token material).

## Failure modes, honestly

| Failure | What happens | Mitigation available here |
|---|---|---|
| Token expires between ticks while the app is down | A request fails; the next tick fixes it | None within this option — this is the reason option 2 exists |
| Refresh endpoint returns 401 (grant dead) | Every tick fails identically | Failure counter → unhealthy → alert; re-consent in the app |
| Worker container stops | Silence, no refreshes | `restart: always`, plus alerting on log absence |
| App container down at tick time | Connection refused, counted as failure | Retry on the next tick |
| Clock skew in the container | Cron fires at unexpected wall times | Container clock follows the host; a 30-minute grid is tolerant |
| Log volume fills | Cron output stops being written | Log rotation on the volume — the operator's responsibility, stated in `deploy.md` |

## Why not just make it smarter

Because that is a different option with a different footprint, and pretending otherwise is how
a 10-line script becomes an untested distributed system. `simple/` is chosen when the app
already owns OAuth and the requirement is "something must call it on a schedule". The moment
the requirement becomes "refresh only when needed, and tell me when it fails", the answer is
[`../../mac/`](../../mac/).
