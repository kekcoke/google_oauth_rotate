# Agentic Orchestration — The Patterns This Project Runs On

A teaching companion to [`orchestration-loop.md`](orchestration-loop.md). That document specifies
*what the loops are*; this one explains *what patterns they are instances of*, so the same shapes
can be recognised and reused elsewhere.

For the portable, project-independent version of these patterns as copy-paste prompts, see
[`prompts/agentic-pattern-library.md`](prompts/agentic-pattern-library.md).

> **How to read this.** Layer 1 is the mental model. Layer 2 is the eight patterns, each mapped to
> real files in this repository. Then anti-patterns, a real worked trace, self-checks, and a
> reference appendix.

---

## Layer 1 — The mental model

An **agentic loop** is any cycle where a model decides what to do next, acts, observes the result,
and repeats until a stop condition. Three things make one useful rather than a novelty:

1. **A falsifiable exit condition.** "Until the tests pass" is falsifiable. "Until it's good" is
   not, and loops without one either stop arbitrarily or never stop.
2. **Externalised state.** If the loop's memory is the conversation, it dies with the context
   window. This repository externalises state into specs, test plans, requirement IDs and git —
   which is why a fresh session can pick up mid-task.
3. **A gate the model cannot talk its way past.** Tests, linters, CI, a human approval. Without
   one, self-assessment is the only quality signal, and self-assessment is generous.

This project runs **two** loops with different tempos and different failure modes:

```text
DEVELOPMENT LOOP                          MAINTENANCE LOOP
(episodic, human-initiated)               (scheduled, alert-driven)

  /spec-new ──► /spec-review                 hourly:  /watch-renew-check
      │              │                       daily:   /token-health
      │              ▼                       weekly:  /reconsent, /deps-audit
      │         findings                     monthly: /rotate-drill
      │              │                            │
      ▼              ▼                            ▼
  /tdd-red ──► /tdd-green ──► /verify        observe → decide → act → alert
      ▲                          │                │
      └──────── not green ───────┘                └── human decides on escalation

  exit: every MUST has a passing test      exit: none — it runs forever
```

The development loop has an end state. The maintenance loop does not, and that difference drives
every design decision about it: no unbounded retries, no silent failures, and an alert on
*absence* of success rather than only on presence of failure.

### Why patterns, not just "an agent"

Naming the pattern tells you its failure mode in advance. A reflection loop fails by
rubber-stamping. A supervisor fails by fanning out forever. Sequential fails by propagating an
early error through every later stage. Knowing which shape you have built is how you know what to
put a guard on.

---

## Layer 2 — The eight patterns

### Pattern inventory

| Pattern | Where it lives here | Status |
|---|---|---|
| [Reflection](#reflection) | `/spec-review`, `token-security-reviewer` | **in use** |
| [Tool use](#tool-use) | per-agent tool allowlists, CI gate scripts | **in use** |
| [Planning](#planning) | `/spec-new`, `oauth-architect`, plan mode | **in use** |
| [ReAct](#react) | `/token-health`, `/watch-renew-check` | **partial** |
| [Sequential](#sequential) | `/tdd-red` → `/tdd-green` → `/verify` | **in use** |
| [Parallel](#parallel) | CI's `docs` and `code` jobs | **partial** |
| [Supervisor](#supervisor) | `/loop-dev` | **in use** |
| [Human-on-the-loop](#human-on-the-loop) | `/reconsent`, `/rotate-drill`, plan approval | **in use** |

**Status is honest.** "Partial" means the shape exists but not in its full form — stated per
pattern below. Nothing here claims capability the repository does not have.

---

### Reflection

**The pattern.** Output is fed back for critique against explicit criteria, then revised. The
critic and the producer are separated so the critique is not self-congratulation.

```text
   produce ──► critique against stated criteria ──► revise
      ▲                                               │
      └──────────── until criteria met ───────────────┘
```

**Here.** [`/spec-review`](../.claude/commands/spec-review.md) checks a spec against nine
mechanical rules, a quality checklist and a set of domain invariants.
[`token-security-reviewer`](../.claude/agents/token-security-reviewer.md) is a second, narrower
critic that gates anything touching tokens, logging or secrets.

**Why it works here.** The criteria are external and mostly mechanical: *does every MUST have a
test ID? does every test ID appear in the applicable test plans? does any requirement contain the
word "correctly"?* A critic checking a checklist cannot flatter itself as easily as one asked
"is this good?".

**Its characteristic failure.** Rubber-stamping. A critic with vague criteria produces "looks
solid, a few minor nits" indefinitely. The countermeasure in this repo is that most checks are
scripts, and the agent's instructions say plainly: *if the spec is sound, say so and name what you
checked — do not invent findings to look thorough.* Both failure directions are named, because a
critic that manufactures findings to seem useful is as bad as one that finds none.

---

### Tool use

**The pattern.** The model acts on the world through a constrained set of tools rather than by
emitting text and hoping.

**Here.** Every agent declares an allowlist, and the allowlists are deliberately *unequal*:

| Agent | Tools | What the restriction encodes |
|---|---|---|
| `spec-writer` | Read, Write, Edit, Grep, Glob | **No Bash.** It writes markdown; it has no business running commands |
| `token-security-reviewer` | Read, Grep, Glob, Bash | **No Write/Edit.** A reviewer that can edit is tempted to fix instead of report |
| `oauth-architect` | Read, Grep, Glob, Bash, WebFetch, WebSearch | **No Write/Edit.** Designs, does not implement |
| `test-author` | Read, Write, Edit, Grep, Glob, Bash | Needs Bash to run tests and confirm they fail correctly |
| `google-api-integrator` | + WebFetch, WebSearch | Explicitly told to fetch current Google docs rather than answer from memory |

**Why it works here.** Least privilege is a design statement, not just a safety measure. Removing
`Write` from the reviewer is what keeps review and repair as separate steps with separate
approvals.

**Its characteristic failure.** Tool sprawl — handing every agent every tool "just in case",
after which role boundaries exist only in the prose. If the security reviewer can edit the file it
just flagged, nothing structurally prevents the finding from disappearing instead of being fixed.

---

### Planning

**The pattern.** Decompose before acting. Produce a plan artifact, review it, then execute it —
rather than discovering scope mid-stream.

**Here.** [`/spec-new`](../.claude/commands/spec-new.md) produces a specification before any code
exists. [`oauth-architect`](../.claude/agents/oauth-architect.md) decides which of the three
options a change belongs in and whether it needs an ADR. Claude Code's plan mode is the same
pattern at the session level — a written plan, approved before execution.

**Why it works here.** The plan artifact is *durable and numbered*. `REQ-002-01` outlives the
session that created it, and a later session can be pointed at it without replaying the reasoning.
This is externalised state doing its job.

**Its characteristic failure.** Planning that never converts to action — successive refinements of
a document nobody executes. The countermeasure is that a spec is not "done" when it reads well; it
is done when every MUST has a test ID, which is a mechanical check that a plan cannot satisfy by
being more eloquent.

The first question `/spec-new` asks is deliberately deflationary: *is this actually a new spec, a
new requirement on an existing one, or an ADR?* Most requests are the second and get treated as
the first.

---

### ReAct

**The pattern.** Interleaved reasoning and acting: observe, reason about what the observation
means, act, observe the result, repeat. The distinguishing feature is that each action is chosen
*after* seeing the previous result, rather than planned up front.

```text
   Thought: the token is 6 days old and the client is in testing mode
   Action:  read the health endpoint
   Observation: state=VALID, refresh_token_age_days=6.2
   Thought: not dead yet, but it will be within 24h
   Action:  emit the day-6 warning, do not re-consent
```

**Here — partial.** The maintenance commands have this shape but are not autonomous loops.
[`/token-health`](../.claude/commands/token-health.md) gathers, interprets against a table of what
each field means, and produces a verdict that names the next action.
[`/watch-renew-check`](../.claude/commands/watch-renew-check.md) explicitly works *down a chain* —
topic exists → Gmail has publish rights → subscription exists → endpoint reachable → receiver
authenticates — where each check is chosen based on the previous result.

**What full ReAct would look like here.** A diagnostic agent that, given "refreshes are failing",
picks its next probe from the previous observation: check Vault seal status → if sealed, stop and
alert; if healthy, check one user's token state → if `DEAD`, check token age to distinguish 7-day
expiry from revocation → and so on. The decision tree exists today in
[`runbooks/`](runbooks/) as prose for a human; nothing executes it autonomously.

**Its characteristic failure.** Thrashing — re-observing the same thing under slightly different
phrasing without narrowing anything. Real ReAct loops need both a step budget and a rule that
each action must *eliminate* a hypothesis.

---

### Sequential

**The pattern.** Stage N's output is stage N+1's input. Order is fixed and each stage has an exit
condition.

**Here.** The development loop's spine:

```text
/spec-new ──► /spec-review ──► /tdd-red ──► /tdd-green ──► /verify ──► commit
   spec        findings        failing      passing        full gate
                               tests        tests
```

Each stage refuses to start until the previous one's exit condition holds.
[`/tdd-green`](../.claude/commands/tdd-green.md) opens by checking that failing tests already
exist and says, in as many words, that implementation without a preceding failing test means
deleting the code and starting from the test.

**Why it works here.** The handoffs are *artifacts*, not conversation: a spec file, a test file, a
green suite. Any stage can be resumed by a different session.

**Its characteristic failure.** Error propagation. A vague requirement becomes a vague test
becomes plausible-but-wrong code, and every downstream stage confirms the mistake. The
countermeasure is that `/tdd-red` is instructed to stop and name the ambiguous test-plan row
rather than guess — pushing the failure back up the chain instead of laundering it.

---

### Parallel

**The pattern.** Independent branches run concurrently, then merge under explicit criteria.

**Here — partial.** CI runs two jobs, `docs and specs` and `lint and test`, concurrently on
separate runners; the merge criterion is that both must pass. Within a session, independent
read-only tool calls are issued in a single batch rather than serially.

**What fuller parallelism would look like here.** Fan-out review: `token-security-reviewer`,
`oauth-architect` and `spec-writer` each reviewing a change simultaneously from their own angle,
with a defined merge rule — for instance, *any security finding blocks; architecture and spec
findings are advisory*. The agents exist and their perspectives genuinely differ; what is missing
is the merge rule, and a fan-out without one is where parallelism turns into noise.

**Its characteristic failure.** No merge criteria. Three reviewers return three opinions and the
orchestrator picks whichever it read last, or averages them into mush. Decide *before* fanning
out what makes a branch authoritative.

Second failure: parallelising work that shares state. Two agents editing the same test plan
concurrently produce a merge conflict at best.

---

### Supervisor

**The pattern.** A coordinator decomposes work, dispatches it, evaluates results, and decides
whether to iterate, escalate or stop.

**Here.** [`/loop-dev`](../.claude/commands/loop-dev.md) drives one spec toward completion:

```text
assess (/option-status)
   │
   ▼
pick the highest-value untested MUST
   │
   ▼
/tdd-red on a small batch ──► /tdd-green ──► /verify ──► commit
   │                                                       │
   └──────────── until every MUST has a passing test ◄──────┘
```

Three properties make it a supervisor rather than a script: it *chooses* what to work on next, it
*batches* (two to five test IDs, not the whole spec), and it has explicit conditions under which
it stops and asks a human.

**Why it works here.** The stop conditions are enumerated rather than left to judgement: spec
ambiguity that would change behaviour, a test that would need a real credential, work requiring
console or Vault access, contradiction with an accepted ADR, and — the telling one — *the same
test failing three iterations for different reasons*, which is a design problem wearing a coding
problem's clothes.

**Its characteristic failure.** Unbounded fan-out. A supervisor that can spawn work without a
budget will, especially when each sub-task looks locally reasonable. The second failure is a
supervisor that reports progress it has not made; `/loop-dev` is instructed that *"three
requirements implemented, two failing" is useful; "good progress" is not.*

---

### Human-on-the-loop

**The pattern.** The system runs autonomously but surfaces decisions a human must make, and waits.
Distinct from human-*in*-the-loop, where a human approves every step.

**Here.**

| Mechanism | The decision reserved for a human |
|---|---|
| [`/reconsent`](../.claude/commands/reconsent.md) | Only a human can complete a browser consent — and the command makes them establish *why* the grant died first |
| [`/rotate-drill`](../.claude/commands/rotate-drill.md) | Confirms which environment before touching credentials |
| Plan approval | The plan is written by the model, approved by the human, then executed |
| `invalid_grant` alerting | The system detects and stops; a human re-authorises |

**Why it works here.** The escalation points are exactly the irreversible or unautomatable ones.
Re-consent needs a browser and an identity decision. Rotation destroys grants. Everything else
runs unattended.

`/reconsent` is the sharpest example of the pattern done properly: before generating any URL it
requires establishing the *cause* of the failure, because "re-consenting a revoked grant without
asking why can walk straight back into a security event." The human is not there to click approve;
they are there because the question is genuinely theirs.

**Its characteristic failure.** Human-*in*-the-way: approval gates on reversible, low-stakes steps
until the human approves reflexively, at which point the gate is theatre. Reserve interruption for
decisions that are irreversible, outward-facing, or need context the system lacks.

---

## Anti-patterns

Five failures that recur across all eight patterns. Each is a design smell rather than a bug.

**1. The loop with no falsifiable exit.**
"Iterate until the code is good." Every iteration produces a plausible reason to continue. Fix:
the exit condition must be checkable by something that is not the model — a test suite, a linter,
a traceability script.

**2. Reflection that rubber-stamps.**
A critic given vague criteria returns "looks good, minor nits" forever. Fix: mechanical criteria
first, judgement second, and explicit permission to return zero findings.

**3. The supervisor with no budget.**
Fan-out without a step, token, or wall-clock limit. Every sub-task is locally justified; the
aggregate is unbounded. Fix: budget declared up front, and a rule for what to do when it is
exhausted (escalate, do not silently truncate).

**4. Parallel branches with no merge rule.**
Three reviewers, three verdicts, no decision procedure. Fix: decide before fanning out which
branch is authoritative for which class of finding.

**5. State that lives only in the conversation.**
The loop's memory is the context window, so it dies with it and cannot be resumed or audited. Fix:
externalise into files with stable identifiers. This repository's `REQ-`/`T-` numbering exists for
exactly this reason — a new session can be told "make `T-002-06` pass" with no other context.

A sixth, specific to this domain: **automation that hides its own failure.** A maintenance loop
that stops running produces silence, and silence looks like success. Hence alerting on the absence
of a successful run, not only on failures ([`REQ-007-07`](../specs/spec-007-observability.md)).

---

## Worked trace — the `/spec-review` run that produced eight fixes

A real run, including the part that went wrong. Composition: **reflection** (the review) →
**planning** (fix design) → **sequential** (fix, verify, commit per finding) → **tool use**
(scripted checks) → **human-on-the-loop** (approval before applying).

### Phase 1 — reflection with mechanical criteria first

`/spec-review` ran nine mechanical checks as shell scripts before any judgement: front matter
validity, every MUST carrying a test ID, one-requirement-per-test, test IDs present in the
applicable plans, fixtures named, index membership, required sections, link resolution.

**All nine passed.** Had the review stopped there — as a rubber-stamping critic would — the answer
would have been "the specs are clean."

### Phase 2 — reflection with domain criteria

The judgement pass found nine defects the mechanical checks structurally could not see:

| # | Defect | Why no script could catch it |
|---|---|---|
| 1 | `REQ-003-09` said "the **verified** ID token's `sub`" but nothing required verification | An adjective is not a requirement. Needs a reader who knows `sub` is the store's primary key |
| 2 | Two requirements gated on "the client is in Testing status" — a fact with no config input | Requires cross-referencing requirements against the environment variable list |
| 3 | `REQ-005-05`: "be idempotent **or documented as not idempotent**" | Syntactically fine; unfalsifiable in substance |
| 4 | `REQ-006-13` referenced a quota figure that was never written down | Needs following the citation and noticing the target is empty prose |
| 5 | `REQ-002-09` used a term defined only in a different option's spec | Requires knowing which options a spec applies to |
| 6 | Compound requirements whose tests covered one clause of three | Needs reading the test against the requirement, not just checking a link |
| 7 | `REQ-003-14` said "ephemeral port"; the deploy doc and `.env.example` pinned a fixed one | Contradiction across three files, none individually wrong |
| 8 | `spec-006` omitted `mac/` from `applies_to` while `mac/` implemented part of it | The traceability check only walked spec → plan, never plan → spec |
| 9 | Duplicate Vault requirements in two specs | Not a rule violation; a drift risk |

**The lesson.** Mechanical checks and judgement checks find disjoint defect classes. A review that
runs only the scripts is fast and finds nothing interesting. A review that runs only judgement
misses the boring structural drift that scripts catch instantly. Both, in that order.

### Phase 3 — human-on-the-loop

The review reported and stopped. Findings 1–8 were approved for fixing; 9 was deferred. The
verdict named *which specs were blocked* rather than giving a global pass/fail — finding 1 was
called out as the one to fix regardless of schedule, being a security gap rather than paperwork.

### Phase 4 — sequential execution, one commit per finding

Eight fixes, each: edit spec → update affected test plans → run the full gate → commit with the
finding number in the message. Requirement IDs were reused rather than renumbered, which is legal
only because every spec is still `draft`.

### Phase 5 — the part that went wrong

Finding 8 needed a new CI check walking plan → spec. First implementation:

```bash
for spec in $(grep -oE 'spec-[0-9]{3}-[a-z-]+\.md' "$plan" | sort -u); do
```

It failed immediately — flagging `simple/`'s plan for "claiming" `spec-001`, which that plan
mentions only in its **Not tested here** section. The grep could not distinguish *"I implement
this"* from *"I explicitly do not implement this, and here is why."*

Narrowing to the "Specs in scope" section was not enough either: prose in that section also names
`spec-001`. The working version reads only table rows:

```bash
in_scope=$(awk '/^## Specs in scope/{f=1;next} /^## /{f=0} f' "$plan" \
           | grep -E '^\| *\[?spec-' \
           | grep -oE 'spec-[0-9]{3}-[a-z-]+\.md' | sort -u)
```

**Why this is the most instructive part of the trace.** The new gate caught its own first version
being wrong, on the first run, before it could be committed as a false constraint. That is the
whole argument for gates the model cannot talk past: the check was written by the same process
that wrote the thing it checks, and it still failed honestly. A self-assessment would have said
"added a CI check ✅".

### Outcome

231 requirements, 240 test IDs, all gates green, eight commits. Two placeholders were left
deliberately empty — the quota table and the loopback-port question — marked *"fill this in"*
rather than populated with invented values.

---

## Self-check

<details>
<summary>1. What makes an exit condition falsifiable, and why does it matter more than the loop's cleverness?</summary>

It can be checked by something that is not the model — a test suite, a linter, a script. It
matters more because a loop with a vague exit either stops arbitrarily or runs forever, regardless
of how well it reasons in between. "Until every MUST has a passing test" is falsifiable; "until
the specs are solid" is not.
</details>

<details>
<summary>2. Why does `token-security-reviewer` deliberately lack Write and Edit?</summary>

So review and repair stay separate steps with separate approvals. A reviewer that can edit is
tempted to fix what it flags, and the finding disappears instead of being reported, decided on,
and recorded.
</details>

<details>
<summary>3. The mechanical checks in the `/spec-review` trace all passed, yet nine
defects existed. What does that say about check design?</summary>

Mechanical and judgement checks find disjoint defect classes. Scripts catch structural drift
instantly and cannot see an unfalsifiable requirement or a missing security precondition. Running
only one kind gives false confidence — cheap and empty, or slow and blind to drift.
</details>

<details>
<summary>4. Distinguish human-on-the-loop from human-in-the-loop, and name the failure of getting
it wrong.</summary>

On-the-loop: the system runs autonomously and surfaces decisions that are genuinely a human's —
irreversible, outward-facing, or needing absent context. In-the-loop: a human approves every step.
Getting it wrong produces approval fatigue: gates on reversible low-stakes steps get approved
reflexively, so the gate that matters is approved reflexively too.
</details>

<details>
<summary>5. Why is "state lives only in the conversation" an anti-pattern here specifically?</summary>

Because the work outlives any one session. Requirement and test IDs, spec files and git history
are the loop's memory; a new session can be told "make `T-002-06` pass" and needs nothing else.
Conversation-only state cannot be resumed, audited, or handed to a different agent.
</details>

<details>
<summary>6. Which pattern is `/loop-dev`, and which single property most distinguishes
it from a shell script running the same commands?</summary>

Supervisor. The distinguishing property is that it *chooses* what to work on next based on an
assessment step (`/option-status`), and can decide to stop and escalate. A script executes a fixed
sequence regardless of what it finds.
</details>

---

## Reference appendix

### File map

| Artifact | Path | Pattern it primarily serves |
|---|---|---|
| `oauth-architect` | [`.claude/agents/oauth-architect.md`](../.claude/agents/oauth-architect.md) | Planning |
| `spec-writer` | [`.claude/agents/spec-writer.md`](../.claude/agents/spec-writer.md) | Planning |
| `test-author` | [`.claude/agents/test-author.md`](../.claude/agents/test-author.md) | Sequential (red stage) |
| `token-security-reviewer` | [`.claude/agents/token-security-reviewer.md`](../.claude/agents/token-security-reviewer.md) | Reflection |
| `google-api-integrator` | [`.claude/agents/google-api-integrator.md`](../.claude/agents/google-api-integrator.md) | Tool use |
| `/spec-new` | [`.claude/commands/spec-new.md`](../.claude/commands/spec-new.md) | Planning |
| `/spec-review` | [`.claude/commands/spec-review.md`](../.claude/commands/spec-review.md) | Reflection |
| `/tdd-red`, `/tdd-green` | [`.claude/commands/`](../.claude/commands/) | Sequential |
| `/verify`, `/option-status` | [`.claude/commands/`](../.claude/commands/) | Tool use |
| `/loop-dev` | [`.claude/commands/loop-dev.md`](../.claude/commands/loop-dev.md) | Supervisor |
| `/reconsent`, `/rotate-drill` | [`.claude/commands/`](../.claude/commands/) | Human-on-the-loop |
| `/token-health`, `/watch-renew-check` | [`.claude/commands/`](../.claude/commands/) | ReAct (partial) |
| CI workflow | [`.github/workflows/ci.yml`](../.github/workflows/ci.yml) | Parallel (partial), Tool use |

### Skills as pattern context

The five skills in [`.claude/skills/`](../.claude/skills/) are not patterns themselves — they are
the durable domain knowledge the patterns operate on, loaded on demand so it does not have to be
rediscovered each session: `spec-driven-development`, `tdd-workflow`, `oauth-token-lifecycle`,
`google-workspace-scopes`, `vault-secrets`.

### Where to go next

- [`orchestration-loop.md`](orchestration-loop.md) — the normative specification of both loops
- [`prompts/agentic-pattern-library.md`](prompts/agentic-pattern-library.md) — the portable prompts
- [`architecture-decisions.md`](architecture-decisions.md) — what these loops were used to build
- [`specs/README.md`](../specs/README.md) — the traceability rules that make the exit conditions
  falsifiable
