# Architecture Decision Records

One file per decision that would be expensive to reverse. Each records the context at the time,
the decision, and what it costs — not just what was chosen.

| ADR | Decision | Status |
|---|---|---|
| [0001](0001-three-deployment-options.md) | Ship three parallel deployment options rather than one | Accepted |
| [0002](0002-expiry-aware-refresh-over-fixed-cron.md) | Expiry-aware refresh is the default; fixed-interval cron only in `simple/` | Accepted |
| [0003](0003-vault-openbao-for-refresh-tokens.md) | Vault/OpenBao holds refresh tokens in `homelab/` | Accepted |
| [0004](0004-incremental-authorization.md) | Request scopes incrementally, store what was granted | Accepted |
| [0005](0005-testing-mode-refresh-token-expiry.md) | Design for 7-day refresh-token expiry as a normal condition | Accepted |
| [0006](0006-bullmq-queue-for-multi-user.md) | BullMQ on Redis for the multi-user worker | Accepted |
| [0007](0007-docker-compose-portable-deploy.md) | Docker Compose everywhere; Kubernetes deferred | Accepted |
| [0008](0008-consent-as-a-one-shot-container.md) | Consent runs in a one-shot container in `mac/`, not on the macOS host | Accepted |

## Writing a new one

Copy the structure of any existing record: `# ADR NNNN — Title`, then **Status**, **Context**,
**Decision**, **Consequences** (good and bad), **Alternatives considered**, **References**.

Never edit an accepted ADR's decision. Supersede it: add a new ADR and mark the old one
`Superseded by ADR-NNNN`, keeping the original reasoning intact so future readers can see what
changed and why.
