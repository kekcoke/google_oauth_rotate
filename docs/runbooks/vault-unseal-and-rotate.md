# Runbook — Vault unseal, and credential rotation

**Applies to:** `homelab/`. `simple/` and `mac/` use an encrypted column with a runtime key;
their key rotation is the last section here.

## Part 1 — Unsealing after a restart

Vault/OpenBao starts **sealed**. A sealed Vault means the worker cannot read refresh tokens, so
no refreshes happen. That is the intended fail-closed behaviour, and it must alert rather than
degrade quietly.

### Symptoms

- Worker health endpoint reports `secrets_backend: unavailable`
- Logs: `Vault is sealed` / HTTP 503 from the Vault address
- Refresh attempts failing for **every** user at once — the giveaway that this is
  infrastructure, not a single grant

### Procedure

1. Confirm the seal status (`vault status` — expect `Sealed: true`, and note the threshold).
2. Provide unseal key shares up to the threshold, **from separate secure locations**, one
   operator action each. Never store all shares together; never paste them into a shell history
   or a chat window.
3. Confirm `Sealed: false` and that the expected KV mount is present.
4. Restart or let the worker re-authenticate. Verify it can read one token path.
5. Verify a real refresh succeeds for one user.
6. Check what was missed during the seal: any token that passed its expiry while Vault was down
   needs a refresh, and any watch renewal in that window needs
   [`watch-renewal.md`](watch-renewal.md).

### Prevention

- Configure auto-unseal if the environment supports it, weighing that it moves trust to the
  auto-unseal key holder.
- Ensure the Vault container restarts on boot with the rest of the stack.
- **Back up the unseal keys and recovery keys.** Losing them loses every stored refresh token,
  which means re-consenting every account. Test the backup by reading it, not by believing in it.
- Alert on Vault seal status directly, not only on refresh failures.

## Part 2 — Rotating the Vault AppRole `secret_id`

Quarterly, and immediately if a worker host is suspected compromised.

1. Generate a new `secret_id` for the worker's AppRole.
2. Deliver it to the worker through the Compose secret mechanism — not into a committed file,
   not as a build argument.
3. Restart the worker and confirm authentication and one successful token read.
4. **Revoke the old `secret_id`.** Rotation without revocation is not rotation.
5. Confirm in the audit log that no reads are still arriving under the old identity.

## Part 3 — Rotating the Google client secret

Annually, and immediately on suspected exposure.

> **This invalidates existing grants.** Every affected account must re-consent. Plan it, or
> discover it at 2am.

1. Create a **new** client secret in the Google Cloud console, keeping the old one for now if
   the console allows both.
2. Write the new secret to Vault (KV v2 keeps the previous version, so a bad write is
   recoverable).
3. Restart the worker and watch for `invalid_grant` across accounts.
4. Re-consent each affected account: [`re-consent.md`](re-consent.md).
5. Delete the old secret in the console **after** every account is re-consented — not before.
6. Record the rotation date and the next due date.

## Part 4 — Rotating the encryption key (`simple/` and `mac/`)

These options hold refresh tokens in an AES-256-GCM encrypted column with the key supplied at
runtime.

1. Introduce the new key alongside the old one, so the store can decrypt with either. The token
   store's ciphertext format must carry a key identifier for this to be possible — a
   requirement in [`spec-001`](../../specs/spec-001-token-store.md).
2. Re-encrypt every row under the new key.
3. Verify a decrypt-and-refresh cycle succeeds.
4. Remove the old key from the environment and from wherever it was stored.
5. If the key was leaked rather than merely aged out, treat the tokens as compromised too:
   [`incident-token-leak.md`](incident-token-leak.md).

## Part 5 — Rehearsal

`/rotate-drill` runs Parts 2–4 against a scratch environment with throwaway credentials.
Monthly. The point is that the real rotation is boring, and that this runbook is discovered to
be wrong during a drill rather than during an incident. If a drill reveals a gap, fix the
runbook in the same session.
