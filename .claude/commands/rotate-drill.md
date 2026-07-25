---
description: Rehearse credential rotation in a scratch environment so the real rotation is boring
argument-hint: [--part approle|client-secret|enc-key|all] [--option mac|homelab]
---

Rehearse credential rotation: $ARGUMENTS

Follow `docs/runbooks/vault-unseal-and-rotate.md`. Monthly. The point is twofold: that the real
rotation is uneventful, and that this runbook is found to be wrong during a drill rather than during
an incident.

## Rules

- **Scratch environment only.** Throwaway credentials, a disposable Vault, a scratch database. Never
  a real client secret, never a real user's token.
- **Confirm the target before touching anything.** State which environment you are about to modify
  and get confirmation. A drill that accidentally rotates production is worse than no drill.
- Follow the runbook **as written**. If a step is unclear or wrong, that is the finding — do not
  improvise past it and call the drill a success.

## Parts

**AppRole `secret_id` (`homelab/`)** — generate a new `secret_id`, deliver it by the Compose secret
mechanism, restart, confirm authentication and one token read, then **revoke the old one** and
confirm no reads still arrive under the old identity. Rotation without revocation is not rotation.

**Client secret** — create the new secret, write it to Vault (KV v2 retains the previous version),
restart, observe `invalid_grant` across accounts, re-consent each, then delete the old secret. Note
that this part is destructive to grants **by design**: the drill's purpose is to measure how long
re-consenting every account actually takes.

**Encryption key (`simple/`, `mac/`)** — load the new key alongside the old, re-encrypt every row,
verify a decrypt-and-refresh cycle, remove the old key. This only works if stored ciphertext carries
a key id; if it does not, that is a defect against REQ-001-05.

**Unseal** — seal Vault, confirm readiness goes false with **one** alert rather than one per user,
confirm no token is served and nothing falls back to a cache, then unseal and confirm recovery
without a worker restart.

## Report

For each part: steps completed, wall time, what broke, and **what the runbook got wrong**. Then fix
the runbook in the same session — an inaccurate runbook discovered and left unfixed is the drill
failing.

Also report the numbers that inform planning: minutes to rotate, minutes per re-consent, and whether
the unseal-key backup was actually read rather than assumed.
