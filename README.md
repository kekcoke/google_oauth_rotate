# google_oauth_rotate

Keeping a Google Workspace OAuth access token fresh, three ways — specified before it is
built.

Access tokens last about an hour. The naive fix is to refresh them constantly; the right fix
is a small background worker that stores the refresh token securely, checks whether the
access token is near expiry, refreshes **only when needed**, saves the new token and expiry,
and hands it to the API client. This repository specifies that worker at three levels of
ambition, and carries the agent/skill/command tooling to build and maintain it.

> **Status: specification-only.** There is no runtime code yet. See
> [`CLAUDE.md`](CLAUDE.md) for how implementation passes are driven from specs.

## Which option do I want?

| If you… | Use | What you get |
|---|---|---|
| Already have an app with a refresh endpoint and just want something poking it | [`simple/`](simple/) | Alpine container running `crond`, one shell script, 30-minute schedule |
| Want one always-on personal worker on your Mac that refreshes intelligently | [`mac/`](mac/) | Docker Desktop container, 5-minute tick, refreshes only inside the skew window, survives restarts |
| Need multiple Google accounts, a queue, and something that lifts to a cloud VM unchanged | [`homelab/`](homelab/) | Compose stack: BullMQ worker, Postgres, Vault/OpenBao, refresh-on-demand with per-user locking |

`simple/` is deliberately dumb — it refreshes on a fixed schedule whether or not the token
needs it. `mac/` fixes that with expiry awareness. `homelab/` adds multi-user isolation,
secret management, and horizontal room to grow. Read
[`docs/architecture-overview.md`](docs/architecture-overview.md) for the comparison in full.

## Start here

1. [`docs/architecture-overview.md`](docs/architecture-overview.md) — the three options side by side
2. [`docs/token-lifecycle.md`](docs/token-lifecycle.md) — the state machine every option must implement
3. [`docs/google-cloud-setup.md`](docs/google-cloud-setup.md) — project, consent screen, client, Pub/Sub
4. [`docs/oauth-scopes.md`](docs/oauth-scopes.md) — what we ask for and why
5. [`docs/security-model.md`](docs/security-model.md) — how the refresh token is protected
6. [`specs/`](specs/) — the shared, numbered requirements each option implements
7. [`docs/orchestration-loop.md`](docs/orchestration-loop.md) — the build and maintenance loops

## Learning

The documents above are normative — they say what must be true. These teach the reasoning behind
them, and link out rather than restating it.

| Document | What it teaches |
|---|---|
| [`docs/architecture-decisions.md`](docs/architecture-decisions.md) | Why three setups exist, what each trades away, and when *not* to use each one |
| [`docs/agentic-orchestration.md`](docs/agentic-orchestration.md) | The eight agentic patterns this project runs on, mapped to real files |
| [`docs/prompts/`](docs/prompts/) | Those patterns as portable, copy-paste prompt modules for any project |
| [`docs/article/`](docs/article/) | A reference-grade case study of the patterns, written for readers outside this project |

Each carries decision matrices, anti-patterns, a worked walkthrough and self-check questions.

## Read this before you build anything

The OAuth client is in **Testing** publishing status, so **Google expires refresh tokens
after 7 days**. This is not a bug to engineer around — it is a property of the environment.
Every option must detect `invalid_grant`, alert, and walk a human through re-consent. The
ways out (Google Workspace internal app, or Production + verification) are documented in
[`docs/production-verification.md`](docs/production-verification.md).

## Repository layout

```text
docs/       architecture, security model, ADRs, runbooks
specs/      shared numbered specs (spec-001 … spec-009) + templates
simple/     Option 1: cron refresh worker
mac/        Option 2: expiry-aware worker on Docker Desktop
homelab/    Option 3: multi-user queue worker, homelab → cloud
.claude/    agents, skills, and slash commands that drive development
```

## Contributing

Specs first, tests second, code third. See [`CLAUDE.md`](CLAUDE.md) for conventions and
[`specs/README.md`](specs/README.md) for the requirement/test traceability rules.
