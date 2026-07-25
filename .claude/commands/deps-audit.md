---
description: Weekly dependency and vulnerability sweep across the three options, with base images
argument-hint: [--option simple|mac|homelab]
---

Audit dependencies and vulnerabilities$ARGUMENTS.

Weekly maintenance. Report; do not upgrade anything without saying what and why first.

## Node dependencies

```sh
npm audit --omit=dev
npm outdated
```

For each finding: is the vulnerable path actually reachable from this project's code? A critical CVE
in an unused code path of a transitive dependency is not the same as one in the OAuth or HTTP path.
Say which it is rather than reporting the severity number alone.

Priority order for this project: anything in the credential path (`google-auth-library`, HTTP
clients, crypto), then the store and queue drivers (`pg`, `bullmq`, Redis clients), then everything
else.

## Base images

```sh
docker compose build --pull
docker scout cves <image>       # or trivy, if that is what is available
```

Check whether base images are still digest-pinned and whether the pinned digest has known CVEs. A
pin is not a licence to stop looking — an unchanged pin with a new critical CVE is worse than an
unpinned image someone is watching.

Also confirm `simple/`'s Alpine pin and the Postgres, Redis and Vault image versions are still
supported releases.

## Secret hygiene, while you are here

```sh
git ls-files | grep -E '\.env$|\.env\.|\.token$|credentials\.json|client_secret' || echo "clean"
```

## Report

| Package or image | Current | Available | Severity | Reachable from our code? | Action |
|---|---|---|---|---|---|

Then:

- **Upgrade now** — reachable vulnerability in the credential, store, or HTTP path
- **Upgrade this week** — reachable elsewhere, or a supported-release deadline approaching
- **Watch** — not reachable, or no fix available
- **Blocked** — a breaking change; say what it breaks

Any upgrade that touches the credential path goes through `/verify` and the
`token-security-reviewer` agent before it is committed. Note also whether the OAuth client is still
in Testing status — the weekly re-consent burden is the other recurring maintenance item, and this
sweep is a reasonable moment to ask whether an exit is worth pursuing yet.
