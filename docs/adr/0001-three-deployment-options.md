# ADR 0001 — Ship three parallel deployment options

**Status:** Accepted

## Context

The source material (`gmail oauth.pdf`) presents three designs for OAuth token rotation —
a Docker cron worker, an expiry-aware background worker, and a persistent-volume deployment
built for cloud portability — and then a separate "for a production system" recommendation
splitting single-user from multi-user. The repository was created with three empty directories
matching those options.

The obvious engineering instinct is to pick the best one and delete the others. That instinct
is wrong here for two reasons: the options serve genuinely different situations (an existing
app that just needs poking; a personal Mac automation; a multi-account service), and the
progression from one to the next is the clearest available explanation of *why* the
sophisticated version is shaped the way it is.

## Decision

Ship all three as independent, individually deployable deliverables:

- `simple/` — Option 1, unconditional cron refresh
- `mac/` — Option 2, expiry-aware worker on Docker Desktop
- `homelab/` — Option 3 plus the production multi-user recommendation

They share one set of numbered specs in `specs/`; each declares which specs apply to it via
`applies_to` frontmatter and carries its own test plan.

## Consequences

### Good

- Each option can be adopted without dragging in the others' infrastructure.
- The shared specs force the options to agree on token semantics, so behaviour is comparable
  rather than divergent.
- The comparison itself is the documentation: `simple/` demonstrates the failure modes that
  justify `mac/`, and `mac/` demonstrates the limits that justify `homelab/`.

### Bad

- Three test plans and three deploy paths to maintain. A change to a shared spec means three
  test plans to revisit — this is the recurring cost.
- Temptation to let `simple/` rot. It is a legitimate deliverable; if it stops being
  maintained, delete it deliberately rather than leaving it broken.
- Readers must be told which option to look at, hence the chooser table in `README.md`.

## Alternatives considered

- **Build only `homelab/`.** Cheapest to maintain, but forces a Postgres/Redis/Vault stack on
  someone whose actual need is a 20-line cron container.
- **One codebase with a configuration switch.** Superficially attractive; in practice the
  three options differ in trigger model, concurrency requirements and storage, so the switch
  becomes a maze of conditionals in exactly the code that must be trustworthy.

## References

- [`docs/architecture-overview.md`](../architecture-overview.md)
