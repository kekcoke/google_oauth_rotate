# Orchestration Loops

Two loops run in this repository: a **development loop** that turns specs into tested code, and a
**maintenance loop** that keeps a deployment alive. They share the same tooling in `.claude/` and
have different cadences and different failure modes.

## Development loop

```text
        ┌──────────────────────────────────────────────────────────────┐
        │                                                              │
   /spec-new ──► /spec-review ──► /tdd-red ──► /tdd-green ──► /verify ─┴─► commit
   (or a new       architect +     failing      minimum        tests, lint,
    REQ in an      security        tests        code           traceability,
    existing spec) agents          only                        secret scan
        ▲                                                              │
        │                     /option-status                           │
        └───────────────── (what is untested?) ◄───────────────────────┘
```

| Step | Command | Exit condition |
|---|---|---|
| Specify | `/spec-new` | Requirements are observable and every MUST has a test id |
| Review | `/spec-review` | Traceability rules pass; no implementation detail in the spec |
| Red | `/tdd-red` | Tests exist and fail **for the intended reason** |
| Green | `/tdd-green` | Tests pass with the minimum code |
| Verify | `/verify` | Full suite, lint, markdownlint, traceability, secret scan all clean |
| Commit | — | Conventional message naming the requirement ids covered |

`/option-status` is the loop's instrument: it reports untested MUSTs, missing test-plan rows, orphan
tests and status lies. Start each iteration there rather than from memory.

### Agents

| Agent | Called for |
|---|---|
| `oauth-architect` | Which option a change belongs in; whether it needs an ADR |
| `spec-writer` | Writing and revising specs, keeping ids traceable |
| `test-author` | The red phase — tests only, never implementation |
| `token-security-reviewer` | **Mandatory** for any change touching tokens, logging, secrets, Vault, Dockerfiles or Compose |
| `google-api-integrator` | Scopes, `watch()`/Pub/Sub, quotas, Google's error semantics |

### Skills

`spec-driven-development` (the id and traceability rules), `tdd-workflow` (red-green-refactor and the
time/concurrency cases this domain needs), `oauth-token-lifecycle` (refresh semantics),
`google-workspace-scopes` (scope selection and consent parameters), `vault-secrets` (secret handling).

### Self-paced mode

`/loop-dev <spec-id>` with the `loop` skill lets a session iterate the cycle on its own cadence:
assess → red a small batch → green → verify → commit → report. It stops and asks a human when the
spec is ambiguous, when a test would need a real credential or a real Google call, when console or
Vault access is required, or when the same test has failed three iterations for different reasons.

Two prohibitions keep an automated loop honest: **never weaken a test to make it pass**, and **never
implement past the failing tests**. Untested code shipped alongside tested code is the worst outcome
of automation.

### CI

`.github/workflows/ci.yml` runs markdownlint, ESLint, the unit suite, the traceability check and a
secret scan on every pull request. It is inert until the repository has a remote. CI enforces the
same gate as `/verify`; the difference is that CI cannot be talked out of it.

## Maintenance loop

Ongoing operation of a deployment. Cadences below assume the OAuth client is in **Testing** status —
several of these jobs exist *only* because of that, and must be **deleted** rather than left running
once an exit is taken ([`production-verification.md`](production-verification.md)).

| Cadence | Job | Command | Watching for |
|---|---|---|---|
| Hourly | Watch renewal | `/watch-renew-check` | `watch()` under 48 h, or missing entirely |
| Daily | Token health | `/token-health` | `DEAD` state, expiry, refresh-token age, sealed backend |
| Daily | Re-consent warning | — (alert) | **Refresh-token age ≥ 6 days** |
| Weekly | Re-consent | `/reconsent` | The 7-day expiry actually landing |
| Weekly | Dependency sweep | `/deps-audit` | Reachable CVEs, unsupported base images |
| Monthly | Rotation drill | `/rotate-drill` | That the runbooks are still correct |
| Quarterly | Backup restore test | — (manual) | That Postgres, Vault and unseal-key backups are real |

### The two clocks

| Clock | Expires | Symptom | Job |
|---|---|---|---|
| Refresh token (Testing mode) | 7 days after issue | `invalid_grant` for that user — loud | Daily health, day-6 warning |
| Gmail `watch()` | ~7 days after registration | **Silence** — no pushes, no errors | Hourly renewal check + silence alarm |

They are independent. A user can hold a healthy token and a dead watch. Conflating them costs a day
of debugging, which is why they have separate jobs and separate runbooks.

### Alerting

Per [`spec-007`](../specs/spec-007-observability.md), these page a human:

- `invalid_grant` on any user → re-consent, naming the user and the runbook
- **No successful refresh within a window** — absence of success, not just presence of failure
- Refresh-token age ≥ 6 days (Testing mode)
- Secret backend unavailable → **one** alert for the backend, never one per user
- No Pub/Sub deliveries over the configured interval
- Queue depth or dead-letter growth (`homelab/`)

The absence alarms matter more than the failure alarms. This system's characteristic failure is a
worker that stopped and said nothing.

### Scheduling

Automate the recurring checks with the `schedule` skill (cron routines) or the host's scheduler.
Whichever runs them, the routine must **report absence** — a maintenance job that silently stops
running reproduces exactly the failure it was meant to catch. Alert on the job not having reported,
not only on it reporting a problem.

## When the loops meet

Maintenance findings feed the development loop. If a runbook step was wrong during a drill, fix the
runbook that session. If an incident revealed an untested path, add the requirement and the test —
`docs/runbooks/incident-token-leak.md` makes this explicit: a leak that could recur unnoticed has not
been fixed. Operational surprises are specification defects, and they are the most valuable input
this repository gets.
