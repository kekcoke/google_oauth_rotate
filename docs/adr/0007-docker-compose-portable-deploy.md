# ADR 0007 — Docker Compose everywhere, Kubernetes deferred

**Status:** Accepted

## Context

The source material's Option 3 is "local Docker + cloud deployment friendly": a persistent
volume, environment-injected credentials, `restart: always`. The requirement is that the
homelab deployment can move to a remote cloud server without a rewrite.

The homelab could plausibly run k3s, and Kubernetes would bring better secret injection,
rolling updates and horizontal scaling. It would also bring manifests, an ingress controller,
a secrets operator and a much larger test surface for what is, at bottom, one worker, one
Postgres, one Redis and one Vault.

## Decision

**Docker Compose is the deployment mechanism for all three options.** `homelab/` ships one
Compose stack that runs unchanged on the homelab box and on a cloud VM; only these differ, and
they differ through configuration, not through a second stack definition:

| Concern | Homelab | Cloud VM |
|---|---|---|
| Secret source | Vault, unsealed locally | Vault, unseal per the runbook |
| TLS / ingress | Existing reverse proxy | Reverse proxy on the VM |
| Backups | Homelab backup regime | Volume snapshots |
| Pub/Sub receiver | Reachable through existing ingress | Public HTTPS on the VM |

Kubernetes is not ruled out; it is deferred, and its migration path is described rather than
built.

## Consequences

### Good

- One artifact to learn, test and document. `docker compose up` is the whole deployment
  interface.
- Genuine portability: the same file, the same volumes, the same service names.
- Matches the source material's intent exactly.

### Bad

- No rolling updates: a worker restart is a brief gap in scheduled sweeps. Acceptable because
  refresh is idempotent and on-demand refresh covers the window.
- No horizontal autoscaling. Multiple worker replicas are possible but must be added
  deliberately, and only work because per-user locking exists.
- Compose secrets are weaker than a Kubernetes secrets operator; Vault carries the load here,
  which is part of why ADR 0003 chose it.
- Host-level concerns (log rotation, volume backups, restart-on-boot) are the operator's job
  and must be spelled out in the deploy docs rather than assumed.

## Alternatives considered

- **Kubernetes / Helm now.** Better long-term scaling and secret story; disproportionate for
  four containers, and it would double this pass's spec surface.
- **Compose now, k8s manifests written speculatively.** Untested manifests are worse than none —
  they rot and mislead. A migration note in this ADR is more honest.
- **Systemd units on the host, no containers.** Loses the parity between homelab and cloud that
  is the whole point of Option 3.

## Migration path if Kubernetes becomes necessary

Triggers: multiple worker replicas needed for throughput, or zero-downtime deploys become a
requirement. Then: keep the worker image unchanged; translate the Compose services to
Deployments plus a StatefulSet for Postgres; replace Compose secrets with the Vault Agent
injector; move repeatable BullMQ jobs to a single-replica Deployment to avoid duplicate
schedules; keep the on-demand guard as the correctness boundary. Write it as ADR 0008 with
measurements that justify it.

## References

- [`homelab/docs/deploy-homelab.md`](../../homelab/docs/deploy-homelab.md)
- [`homelab/docs/deploy-cloud.md`](../../homelab/docs/deploy-cloud.md)
