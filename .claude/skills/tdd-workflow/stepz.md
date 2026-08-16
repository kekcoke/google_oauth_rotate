Here are the six steps as executable sequences. Precondition: merge the two open PRs first (gh pr merge 1 --squash, then gh pr merge 2 --squash) so main carries the plan.

Step 1 — Settle the specs · Opus

A ratchet with no machine oracle. Never delegate this one.

git checkout main && git pull
git checkout -b spec/accept-mac-specs
claude --model opus

/model opus                      # confirm, don't assume
Read mac/IMPLEMENTATION.md. Execute Step 1 only — no code.
Resolve B3, B5, B6, B9; promote the 10 mac specs draft→accepted;
write docs/adr/0008-consent-as-a-one-shot-container.md.

/spec-review spec-200
/spec-review spec-004
/option-status --option mac

git add -A && git commit -m "spec(mac): accept the ten mac-applicable specs"
gh pr create --base main --title "Accept the mac specs, resolving B1-B12"
gh pr merge --squash --delete-branch

Gate: grep -l 'status: draft' specs/spec-*.md mac/specs/spec-*.md returns nothing.

Step 2 — Scaffold · Sonnet, with an Opus interrupt

Every decision is already made in IMPLEMENTATION.md; docker build is the oracle.

git checkout main && git pull
git checkout -b chore/mac-scaffold
claude --model sonnet

Read mac/IMPLEMENTATION.md Step 2. Create the scaffolding exactly as
specified: package.json, Dockerfile, .dockerignore, docker-compose.yml,
eslint.config.js, .c8rc.json. Do not invent dependencies.

Then switch in the same session for the one judgment call — the REFRESH_ABSENCE_WINDOW_MS lid-close false alarm:

/model opus
Close the B8 open question: absence alerts fire every time the lid closes
overnight. Implement the resume-event suppression, or write the superseding
requirement. Decide and justify.

cd mac && npm ci && npx eslint . && node --test
git add -A && git commit -m "chore(mac): scaffold the package, image and lint rules"
claude --model opus
/fast                            # Opus with faster output; same model

Read mac/IMPLEMENTATION.md Step 3. Critical path only — ~30 test IDs.
Build all 32 fixtures now; steps 4-6 reference them by name.

/tdd-red spec-009 --option mac --tests T-009-01,T-009-02
/tdd-green spec-009 --option mac
/tdd-red spec-001 --option mac --tests T-001-01,T-001-02,T-001-03
/tdd-green spec-001 --option mac
/tdd-red spec-002 --option mac --tests T-002-01,T-002-02,T-002-03,T-002-04,T-002-05
/tdd-green spec-002 --option mac

If it gets long, split at the consent boundary — never the Docker boundary. Then the milestone:

./scripts/consent.sh --scopes gmail.readonly      # real browser, real Google
./scripts/up.sh
curl -s localhost:8080/health | jq                # poll: secondsUntilExpiry must drop, then jump
docker compose logs gmail-worker | grep refreshed # exactly one refreshed=true

git commit -m "feat(mac): walking skeleton refreshing a real token"
gh pr create --base main

Before merging, review on Opus: /code-review high and a token-security-reviewer pass.

Steps 4–6 — Backfill · worktrees, mixed models

This is where worktrees pay off, because the dependency graph is a DAG with parallel layers, not a chain:

layer 0:  001, 009          ← no dependencies, fully parallel
layer 1:  004, 007, 008     ← depend only on 001
layer 2:  002 → 003 → 005 → 006 → 200

Run layer 0 and layer 1 concurrently in separate worktrees, each its own terminal:

git worktree add ../oauth-001 -b test/mac-spec-001-backfill
git worktree add ../oauth-007 -b test/mac-spec-007-backfill
git worktree add ../oauth-008 -b test/mac-spec-008-backfill
git worktree list

Per worktree, the model splits within the TDD cycle — this is the inversion from last turn:

cd ../oauth-001 && claude --model opus

/option-status --option mac              # cheap state reload, not a spec re-read
/tdd-red spec-001 --option mac --tests T-001-10,T-001-11,T-001-12

Red is where a wrong test passes silently, so it stays on Opus for time, concurrency and negative cases. Then drop for green:

/model sonnet
/tdd-green spec-001 --option mac
/verify --option mac
/clear                                   # between test batches — the real token lever

For the 19 fully-elaborated spec-200 rows, red is transcription — run those on Sonnet throughout.

Merge each worktree independently, then clean up:

git add -A && git commit -m "test(mac): complete spec-001 coverage (T-001-04..18)"
git push -u origin test/mac-spec-001-backfill
gh pr create --base main --title "spec-001 token store: full coverage"
gh pr merge --squash --delete-branch

cd ../google_oauth_rotate
git worktree remove ../oauth-001
git worktree prune

Serialize layer 2 (002 → 003 → 005 → 006 → 200) in the main checkout — those genuinely chain.

For a long unattended backfill, /loop-dev self-paces:

claude --model sonnet
/loop 30m /loop-dev spec-007 --option mac

Cheat sheet

┌──────────┬────────────────────────────┬──────────────┬─────────────────┐
│   Step   │           Branch           │    Model     │    Parallel?    │
├──────────┼────────────────────────────┼──────────────┼─────────────────┤
│ 1 Specs  │ spec/accept-mac-specs      │ Opus         │ No — ratchet    │
├──────────┼────────────────────────────┼──────────────┼─────────────────┤
│ 2        │                            │ Sonnet +     │ Runs alongside  │
│ Scaffold │ chore/mac-scaffold         │ Opus         │ the console     │
│          │                            │ interrupt    │ gate            │
├──────────┼────────────────────────────┼──────────────┼─────────────────┤
│ 3        │ feat/mac-walking-skeleton  │ Opus         │ No — the        │
│ Skeleton │                            │              │ milestone       │
├──────────┼────────────────────────────┼──────────────┼─────────────────┤
│ 4        │ test/mac-spec-NNN-backfill │ Opus red /   │ 3 worktrees     │
│ Backfill │                            │ Sonnet green │ (001, 007, 008) │
├──────────┼────────────────────────────┼──────────────┼─────────────────┤
│ 5        │ feat/mac-worker-complete   │ Mixed        │ Partly — 004    │
│ SPEC-200 │                            │              │ and 006 split   │
├──────────┼────────────────────────────┼──────────────┼─────────────────┤
│ 6        │ feat/mac-adapters          │ Sonnet       │ 4 worktrees,    │
│ Adapters │                            │              │ one per adapter │
└──────────┴────────────────────────────┴──────────────┴─────────────────┘

Three commands do most of the token work: /clear between batches, /option-status --option mac instead of re-reading 1,916 lines of spec, and node --test --test-name-pattern 'T-001-1[012]' instead of the full suite while iterating.

One caution on worktrees: they share .git but not node_modules. Each needs its own cd mac && npm ci, and running three docker compose up stacks against the same named volume will trip the single-instance guard by design — give each worktree its own project name (docker compose -p mac-001) or keep containers to one worktree at a time.