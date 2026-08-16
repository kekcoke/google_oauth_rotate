# ADR 0008 — Consent runs in a one-shot container, not on the macOS host

**Status:** Accepted

## Context

`mac/` stores its token record on a Docker **named volume**, and its `DATABASE_URL` is
`sqlite:///data/tokens.db` — a container-absolute path. The consent flow
([spec-003](../../specs/spec-003-consent-flow.md)) has to write the first record into that store,
because until it does the worker has nothing to refresh.

The deploy documentation described consent as a host process: run `node scripts/consent.js` on
macOS and it "writes the record to the same volume the container uses". That cannot work. A named
volume lives inside Docker Desktop's Linux VM; a macOS process has no path to it, and `/data`
resolves on the host to a directory that is not the store. The consent helper would create a
second, empty SQLite database somewhere on the host, report success, and the worker would go on
reporting "not consented" forever.

The symptom was already known — `mac/docs/deploy.md` carried a troubleshooting row reading
"Record written to a different store than the container mounts" — but it was filed as an operator
mistake rather than as the design being impossible. It is the largest blocker in the option: no
consent means no token, and no token means nothing else in `mac/` can be demonstrated.

Three properties constrain the fix:

- The browser runs on the host, so the loopback redirect must land on a host port. REQ-003-14
  requires the redirect URI to match the registered one byte-for-byte, so the URI must not change.
- The refresh token is encrypted with `TOKEN_ENC_KEY`, which reaches the worker by runtime
  injection from the Keychain (REQ-200-15). Whatever writes the record needs the same key, through
  the same seam.
- Writing that record is the most security-sensitive operation in the system. It must not acquire
  a new attack surface in exchange for convenience.

## Decision

**Consent runs as a second Compose service on the same image, behind a profile, mounting the same
volume.** It is a one-shot process: it starts, serves one callback, writes one record, and exits.

```yaml
consent:
  profiles: ["consent"]
  build: .
  ports: ["127.0.0.1:8765:8765"]
  volumes: [tokens:/data]
  command: ["node", "src/consent-cli.js"]
```

`mac/scripts/consent.sh` reads the key from the Keychain, runs
`docker compose run --rm --service-ports consent --scopes …`, and opens the URL the container
prints. The `consent` profile keeps the service out of an ordinary `docker compose up`.

The consent container is **not a worker**. It does not acquire the single-instance guard
(REQ-200-06) and may run while a worker holds it; concurrent access to the store is serialised by
the store's transaction semantics (REQ-001-07). This is recorded in SPEC-200's *Definitions*.

## Consequences

### Good

- **The redirect URI does not change.** The browser still hits
  `http://localhost:8765/oauth2callback`; Docker forwards the published port into the container.
  REQ-003-14 is untouched and the registered client needs no edit.
- **One store, one code path.** A single `DATABASE_URL`, one store implementation and one
  encryption path are valid in both processes. The "record written to a different store" failure
  mode stops existing rather than being documented.
- **No bind mount, so no SQLite over VirtioFS.** File locking across a macOS bind mount is exactly
  where the atomic upsert (REQ-001-07) and the instance guard (REQ-200-06) would rot quietly —
  passing tests, corrupt data.
- **One image, so the existing image checks already cover consent.** T-008-01 and T-008-15 assert
  no credential is baked into the image; they now assert it for the consent path for free.
- **One secret seam.** `TOKEN_ENC_KEY` reaches the consent container exactly as it reaches
  `scripts/up.sh` — Keychain to environment to `docker compose`. One seam, one test (T-200-16),
  one thing to document.

### Bad

- A second Compose service and a profile flag the operator has to remember. `docker compose up`
  alone will not run consent, which is correct but is one more thing to get wrong.
- The consent URL is printed by a container rather than opened directly by the process that built
  it, so the wrapper script has an extra step that can fail on its own.
- `docker compose run --service-ports` is a less familiar invocation than `node script.js`, and
  its failure messages are Docker's rather than the application's.
- The key is visible in the environment of the `docker compose run` process while it runs — the
  same honest tradeoff already accepted for `scripts/up.sh`, and part of why
  [`homelab/`](../../homelab/) uses Vault instead.

## Alternatives considered

- **An HTTP endpoint on the worker that accepts a written record.** Rejected: it adds a new,
  unauthenticated write surface to the most sensitive operation in the system, on a process whose
  whole security posture is that it is reachable from loopback only and accepts no input.
- **`docker cp` the host-written database into the volume.** Rejected: the copy is not atomic, it
  can overwrite a record the worker is mid-write on, and the host still needs both the encryption
  key and a working SQLite writer — so it keeps every cost of the host approach and adds a race.
- **A bind mount instead of a named volume, so the host can write the file directly.** Rejected:
  it puts SQLite's locking on VirtioFS, which is the one filesystem property this option cannot
  afford to be wrong about, and it contradicts the standing rule that state lives on a named
  volume.
- **Run the whole worker on the host, no container.** Rejected: it discards `restart: always`,
  the sleep/wake behaviour the option exists to prove, and the parity with
  [ADR 0007](0007-docker-compose-portable-deploy.md).

## References

- [`mac/IMPLEMENTATION.md`](../../mac/IMPLEMENTATION.md) — decision B2, and the step that applies it
- [`mac/specs/spec-200-expiry-aware-worker.md`](../../mac/specs/spec-200-expiry-aware-worker.md)
- [`specs/spec-003-consent-flow.md`](../../specs/spec-003-consent-flow.md) — REQ-003-14
- [`mac/docs/deploy.md`](../../mac/docs/deploy.md)
- [ADR 0007](0007-docker-compose-portable-deploy.md) — Compose as the deployment mechanism
