# `homelab/` — Deploy to a cloud VM

**The same Compose file.** That portability is the point of Option 3, and it is why Kubernetes was
deferred ([ADR 0007](../../docs/adr/0007-docker-compose-portable-deploy.md)). Read
[`deploy-homelab.md`](deploy-homelab.md) first — everything there applies. This document covers
only what differs.

## What differs

| Concern | Homelab | Cloud VM |
|---|---|---|
| Vault unseal | Manual, keys held locally | Manual, or auto-unseal via the provider's KMS — a deliberate trust decision |
| TLS / ingress | Existing reverse proxy | Reverse proxy on the VM, or a managed load balancer; a DNS name you control |
| Push receiver reachability | Through existing ingress | Public HTTPS on the VM — now genuinely exposed to the internet |
| Backups | Homelab backup regime | Volume snapshots plus off-instance copies |
| Egress | LAN, then internet | Provider egress, possibly metered |
| Secrets at rest | Local disk | Provider disk encryption, plus Vault on top |
| Firewall | Homelab network | Provider security groups — **default-deny inbound except 443** |
| Cost | Electricity | Per-hour, and snapshots are not free |

## Provisioning

Modest sizing: the worker is I/O-bound on HTTPS calls, not CPU-bound. 2 vCPU / 4 GB comfortably
runs the whole stack for tens of users; Postgres and Vault are the memory consumers.

1. Create the VM with full-disk encryption enabled.
2. Security group: **deny all inbound**, then allow `443` from anywhere (Pub/Sub push needs it)
   and SSH only from your own address or, better, through the provider's bastion or an identity-aware
   proxy.
3. Never expose `5432` (Postgres), `6379` (Redis), or `8200` (Vault). Compose keeps them on the
   internal network; do not add published ports "temporarily".
4. Install Docker and Compose v2. Enable the Docker service at boot so `restart: always` means
   something after a host reboot.
5. Set up NTP. Every expiry comparison depends on the clock.
6. Attach a separate volume for `pgdata` and `vaultdata` so it survives instance replacement.

## The exposure change worth pausing on

On the homelab the push receiver sits behind existing ingress. On a cloud VM it is a public HTTPS
endpoint that anyone can reach. The receiver's OIDC verification
([spec-006](../../specs/spec-006-watch-pubsub.md) REQ-006-04) stops being a good practice and
becomes the only thing between the internet and your history-sync job queue.

Before going live, verify from an unrelated network:

```sh
curl -s -o /dev/null -w '%{http_code}\n' https://<host>/pubsub/push -d '{}'   # expect 401/403
curl --max-time 3 https://<host>:8200/v1/sys/health                          # expect no route
curl --max-time 3 <host>:5432                                                # expect no route
```

Add a rate limit at the proxy. Pub/Sub retries aggressively and an unauthenticated flood should
not reach Node.

## Vault on a cloud VM

Manual unseal means every reboot needs a human, and reboots on a cloud VM are more common than in
a homelab. Auto-unseal via the provider's KMS removes that toil but moves trust to the KMS key
holder — which on a personal deployment is the same person, and is usually the right trade.
Decide explicitly and write it down; do not drift into a root token in the environment because
unsealing was inconvenient at the wrong moment.

Whichever you choose: **unseal and recovery keys must be backed up off the instance.** A snapshot
of an encrypted volume whose keys were only ever on that volume is not a backup.

## Migration from homelab to cloud

Order matters, and the last step is the one people forget.

1. Bring the cloud stack up empty and verify health.
2. Stop the homelab worker so nothing writes during the copy.
3. `pg_dump` and restore into the cloud Postgres.
4. Move Vault contents deliberately — either a Vault-level migration or re-write each
   `secret/oauth/<user>/refresh` entry. Never through a shell history or a log.
5. Update the OAuth client's redirect URI in the Google Cloud console to the new host. It must
   match exactly.
6. Update the Pub/Sub push subscription's endpoint URL and OIDC audience.
7. Re-register `watch()` for each user; watches are tied to the topic, and the receiver moved.
8. Start the cloud worker with `IS_SCHEDULER=true` on exactly one replica.
9. Verify per user: token state, one successful refresh, one real push received.
10. Only then decommission the homelab stack — and revoke anything it still holds.

Expect at least one `invalid_grant` during a move if the client secret was rotated on the way.
That is a re-consent, not a mystery.

## Ongoing differences

- **Snapshots are not backups** until a restore has been tested. Quarterly, restore into a
  scratch instance and read a token back.
- **Watch the bill.** A tight polling interval on a metered egress link is a cost bug as well as a
  quota bug.
- **Patch the host.** In a homelab this slides; on a public IP it does not.
- **Alert on reachability from outside.** A firewall rule change that exposes Postgres is
  survivable if it is noticed within minutes.

## What would justify Kubernetes

Multiple worker replicas for throughput, or a zero-downtime deploy requirement. Neither is true
of a personal or small multi-user deployment. If it becomes true, ADR 0007 describes the
migration path — and it starts with measurements, not with a chart.
