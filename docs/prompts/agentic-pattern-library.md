# Agentic Pattern Library

Eight modular prompt blocks — four core workflow patterns, four orchestration patterns — plus a
shared preamble that goes above whichever you pick.

**Portable by design.** Nothing here depends on this project, on a particular model, or on a
particular tool. Placeholders are written `{{LIKE_THIS}}`. Claude Code specifics are quarantined
in one section at the end so the rest travels unchanged.

---

## How to assemble a prompt

```text
┌──────────────────────────────────┐
│  BLOCK 0 — shared preamble       │   always
│  role · context · constraints    │
│  output contract · STOP condition│
├──────────────────────────────────┤
│  BLOCK N — one pattern module    │   usually one; sometimes composed
├──────────────────────────────────┤
│  Your task                       │
└──────────────────────────────────┘
```

Fill in every placeholder. An unfilled `{{STOP_CONDITION}}` is the single most common cause of a
loop that runs forever or stops arbitrarily.

## Selection heuristic

Find the row that matches the *shape* of your task, not its subject matter.

| If the task… | Use | Why |
|---|---|---|
| Has a quality bar that is hard to hit first try | [Reflection](#1--reflection) | Critique against explicit criteria beats one long attempt |
| Requires reading, running or changing real things | [Tool use](#2--tool-use) | Constrain what it can touch before it touches anything |
| Is a diagnosis — the next step depends on what you find | [ReAct](#3--react) | Each action is chosen after the previous observation |
| Is large, and scope is uncertain | [Planning](#4--planning) | Produce a reviewable artifact before doing work |
| Has fixed stages with clear handoffs | [Sequential](#5--sequential) | Exit conditions between stages stop errors propagating |
| Splits into genuinely independent parts | [Parallel](#6--parallel) | Concurrency, but only with a merge rule decided up front |
| Needs work chosen, dispatched and judged repeatedly | [Supervisor](#7--supervisor) | A coordinator that can also decide to stop |
| Touches anything irreversible or outward-facing | [Human-on-the-loop](#8--human-on-the-loop) | Reserve interruption for decisions that are genuinely a human's |

Two rules of thumb. **Default to the simplest pattern that fits** — most tasks that look like they
need a supervisor need a sequential chain. And **compose deliberately**: patterns stack, but each
added layer multiplies cost and failure surface.

---

## BLOCK 0 — Shared preamble

Goes above every pattern module.

```text
ROLE
You are {{ROLE}}. {{ONE_SENTENCE_ABOUT_EXPERTISE_THAT_CHANGES_BEHAVIOUR}}

CONTEXT
{{WHAT_EXISTS_ALREADY}}
{{WHERE_TO_LOOK_FIRST}}
{{WHAT_HAS_ALREADY_BEEN_DECIDED_AND_IS_NOT_UP_FOR_DEBATE}}

CONSTRAINTS
- {{HARD_CONSTRAINT_1}}
- {{HARD_CONSTRAINT_2}}
- Do not {{THE_THING_THAT_WOULD_QUIETLY_RUIN_THIS}}

OUTPUT CONTRACT
Produce exactly: {{ARTIFACT_SHAPE}}
Do not produce: {{COMMON_OVERREACH}}

STOP CONDITION
Stop when {{FALSIFIABLE_CONDITION_CHECKABLE_BY_SOMETHING_OTHER_THAN_YOU}}.
If you cannot satisfy it, stop and say precisely what is blocking you.
Do not continue past this condition to add improvements nobody asked for.

REPORTING
State what you did, what you did not do and why, and anything you are uncertain about.
Report failures with their actual output. Never describe work as complete when it is partial.
```

### Why each field earns its place

| Field | The failure it prevents |
|---|---|
| ROLE | Generic output that ignores domain constraints |
| CONTEXT: already decided | Re-litigating settled decisions every session |
| CONSTRAINTS: "do not…" | The specific quiet ruin — usually a rewrite, a scope grab, or a deleted test |
| OUTPUT CONTRACT | Essays where a diff was wanted, and vice versa |
| STOP CONDITION | Loops that never converge, or stop for no reason |
| REPORTING | "Done ✅" over a half-finished task |

---

## Core workflow patterns

## 1 — Reflection

**Use when** the quality bar is hard to hit first try, the criteria can be stated, and a wrong
answer is expensive: specs, security review, API design, anything a person will rely on.

**Skip when** the task is mechanical, or when a test suite already provides the critique.

### Prompt

```text
You will work in two roles, separated. Do not blend them.

ROLE A — PRODUCER
Produce {{ARTIFACT}} meeting {{REQUIREMENTS}}.

ROLE B — CRITIC
Evaluate the artifact against these criteria, in this order:

  MECHANICAL (check these first; they are objective)
  1. {{CHECKABLE_RULE_1}}
  2. {{CHECKABLE_RULE_2}}

  JUDGEMENT (only after the mechanical checks)
  3. {{QUALITY_CRITERION_1}}
  4. {{QUALITY_CRITERION_2}}

For each finding: the location, the concrete defect, and the smallest fix.
Order findings worst-first.
If the artifact is sound, say so and name what you checked.
Do not invent findings to appear thorough. Do not soften a real finding to appear agreeable.

REVISION
Address findings worst-first. If you disagree with one, say why rather than silently skipping it.

ITERATION BUDGET: {{N}} rounds. Exit when the mechanical checks pass and no judgement finding
is rated blocking — or when the budget is exhausted, in which case report what remains.
```

### Parameters

| Placeholder | Guidance |
|---|---|
| `{{CHECKABLE_RULE_N}}` | Must be objectively true or false. "Every requirement has a test ID" — not "requirements are clear" |
| `{{QUALITY_CRITERION_N}}` | Name the *specific* failure to hunt: unfalsifiable wording, missing prohibitions, contradictions across files |
| `{{N}}` | 2–3. Beyond that, returns fall off sharply |

### Failure signatures and exit criteria

| Signature | What it means | Response |
|---|---|---|
| "Looks solid, a few minor nits" every round | Criteria too vague | Replace judgement criteria with mechanical ones |
| Finding count rises each round | Critic is manufacturing work | Cap rounds; require severity ratings |
| Revisions introduce new defects | Rewriting rather than fixing | Constrain to smallest-fix-per-finding |
| Critic praises the producer's reasoning | Roles have blended | Split into separate calls with separate context |

**Exit:** mechanical checks pass, no blocking judgement finding, budget not exceeded.

### Anti-patterns and when to switch

- **Self-review in one pass** ("write it, then check it") — the model has already committed to its
  answer. Separate the calls.
- **Reflection as a substitute for tests.** If the criteria are executable, execute them; use
  reflection for what cannot be executed.
- **Switch to [Sequential](#5--sequential)** when critique keeps finding the same class of defect —
  that is a missing stage, not a critique problem.

### Cost, latency and model choice

Roughly 2–3× a single pass. Critic quality dominates the outcome, so spend the stronger model
there; the producer can often be a tier lower. Latency is serial by nature — do not expect to
parallelise producer and critic.

### Worked micro-example

```text
ROLE A — PRODUCER
Produce a migration plan for splitting the `users` table into `users` and `user_profiles`.

ROLE B — CRITIC
  MECHANICAL
  1. Every step names the table and columns it touches.
  2. Every destructive step has a preceding backup or reversible step.
  3. Every step states whether it holds a lock and for how long.

  JUDGEMENT
  4. Does any step break running application code between deploys?
  5. Is there a step that cannot be rolled back, and is it flagged as such?

ITERATION BUDGET: 2 rounds.
```

Criterion 4 is the one that finds real defects — it requires holding two deploy states in mind at
once, which is exactly what a mechanical check cannot do.

---

## 2 — Tool use

**Use when** the task requires reading, running or changing real things. Which is most real tasks.

**Skip when** the work is pure text transformation with no external state.

### Prompt

```text
TOOLS AVAILABLE
{{TOOL_1}} — use for {{PURPOSE}}. Do not use for {{MISUSE}}.
{{TOOL_2}} — use for {{PURPOSE}}.

TOOLS DELIBERATELY WITHHELD
{{WITHHELD_TOOL}} — because {{REASON}}. If you need it, stop and say so.

RULES
- Prefer the most specific tool that fits. Do not reach for a shell when a dedicated tool exists.
- Verify before mutating: read the target before you overwrite or delete it.
- Batch independent calls; do not serialise work that has no dependency between steps.
- Never pass {{SENSITIVE_THING}} to any tool that logs, echoes, or transmits.
- After a failed call, diagnose before retrying. Do not retry the same call unchanged.

BUDGET
At most {{N}} tool calls. If you approach it, stop and report what you learned and what remains.
```

### Parameters

| Placeholder | Guidance |
|---|---|
| `{{WITHHELD_TOOL}}` | The most valuable field. Withholding `write` from a reviewer keeps review and repair separate |
| `{{SENSITIVE_THING}}` | Credentials, tokens, personal data — name them explicitly; "be careful" is not a rule |
| `{{N}}` | Generous but finite. Its job is to catch runaway exploration, not to ration normal work |

### Failure signatures and exit criteria

| Signature | What it means | Response |
|---|---|---|
| Shell used where a dedicated tool exists | Tool descriptions too thin | Add "use for / do not use for" per tool |
| Same failing call retried verbatim | No diagnosis step | Add the explicit no-blind-retry rule |
| Reads the same file repeatedly | Not tracking what it knows | Ask for a running summary of established facts |
| Mutation with no preceding read | Verify-before-mutate missing | Restate; consider withholding the mutating tool |
| Budget exhausted with nothing concluded | Task under-specified | Stop. The problem is upstream of the tools |

**Exit:** the task's own completion criterion, or budget reached with a report of findings.

### Anti-patterns and when to switch

- **Tool sprawl.** Every agent gets every tool "just in case", after which role boundaries exist
  only in prose.
- **The withheld tool that is not really withheld** — no `write` tool, but shell access.
- **Switch to [ReAct](#3--react)** when the sequence of calls cannot be known in advance.
- **Switch to [Human-on-the-loop](#8--human-on-the-loop)** when a call would be irreversible or
  outward-facing.

### Cost, latency and model choice

Cost is dominated by tool *output* volume, not by reasoning — a single unfiltered log dump can
outweigh the whole conversation. Filter at the source. Tool selection is easy for mid-tier models;
interpreting messy output is not.

### Worked micro-example

```text
TOOLS AVAILABLE
read_file  — inspect source. Do not use to enumerate a directory; use list_files.
list_files — enumerate paths.
run_tests  — execute the suite. Read output fully before concluding.

TOOLS DELIBERATELY WITHHELD
write_file — this is a diagnosis task. If a fix is needed, describe it; do not apply it.
shell      — withheld for the same reason; run_tests covers the legitimate need.

RULES
- Never include environment variable values in your report; names only.
- After a failed test run, read the failure output before forming a hypothesis.

BUDGET: 20 tool calls.
```

---

## 3 — ReAct

**Use when** the next action genuinely depends on what the previous one revealed: debugging,
incident response, exploring an unfamiliar system.

**Skip when** the steps are knowable up front — that is [Sequential](#5--sequential), and ReAct
will be slower and chattier for no benefit.

### Prompt

```text
Work in explicit cycles. Show each cycle.

  THOUGHT:      what you currently believe, and which hypothesis you are testing
  ACTION:       the single next step, chosen to ELIMINATE a hypothesis
  OBSERVATION:  what actually happened — quote it, do not paraphrase
  UPDATE:       which hypotheses are now eliminated, and what remains

HYPOTHESES (start here; add only with a reason)
1. {{HYPOTHESIS_1}}
2. {{HYPOTHESIS_2}}
3. {{HYPOTHESIS_3}}

RULES
- Every action must be capable of ruling something out. If it cannot, do not take it.
- Never restate an observation as a new thought without narrowing the hypothesis set.
- If two consecutive cycles eliminate nothing, stop and say what you would need to make progress.

BUDGET: {{N}} cycles.
STOP when the hypothesis set is reduced to one and confirmed, or the budget is reached.
Report the surviving hypothesis, the evidence, and everything you ruled out.
```

### Parameters

| Placeholder | Guidance |
|---|---|
| `{{HYPOTHESIS_N}}` | Seed with 3–5 plausible causes. An unseeded ReAct loop wanders |
| `{{N}}` | 5–10 for diagnosis. If it needs more, the hypothesis set was wrong |

### Failure signatures and exit criteria

| Signature | What it means | Response |
|---|---|---|
| Thoughts restate observations | Not narrowing | Enforce the UPDATE step naming eliminations |
| Hypothesis set grows every cycle | Speculating, not testing | Require a reason to add |
| Same probe under different phrasing | Thrashing | Stop; the instrumentation is insufficient |
| Confident conclusion, thin evidence | Premature convergence | Require the eliminating observation to be quoted |

**Exit:** one hypothesis survives *and* is positively confirmed — not merely last remaining.

### Anti-patterns and when to switch

- **ReAct where a checklist would do.** If a runbook exists, follow it.
- **Acting without the reasoning shown** — you lose the ability to see where it went wrong.
- **Switch to [Planning](#4--planning)** once the cause is known: diagnosis and repair are
  different tasks with different failure modes.

### Cost, latency and model choice

The most expensive core pattern: every cycle carries the full history. Costs grow super-linearly
with cycle count, which is the real argument for the budget. Needs a strong model — cheaper models
thrash. Consider summarising eliminated hypotheses rather than carrying full history.

### Worked micro-example

```text
HYPOTHESES
1. The scheduled job is not running at all.
2. It runs but its credential is expired.
3. It runs and succeeds, but writes somewhere nothing reads.
4. It runs and fails silently because errors are swallowed.

THOUGHT: (2) and (4) are cheapest to eliminate — both visible in logs.
ACTION: read the last 200 log lines for the job's identifier.
OBSERVATION: 14 entries, all "completed in 1.2s", none with an error.
UPDATE: (1) and (4) eliminated — it runs, and it is not silently failing.
        (2) unlikely: an expired credential would surface as an error.
        (3) now the leading hypothesis.

THOUGHT: test (3) by comparing where it writes against where the reader looks.
ACTION: ...
```

Two hypotheses eliminated by one action, because the action was chosen to discriminate rather than
to gather.

---

## 4 — Planning

**Use when** scope is uncertain, the work is large enough that direction matters more than speed,
or several people must agree before work starts.

**Skip when** the change is small and obvious. A plan for a typo fix is theatre.

### Prompt

```text
Produce a plan. Do not implement anything yet.

INVESTIGATE FIRST
Read {{WHERE_TO_LOOK}} before proposing anything.
Identify existing {{FUNCTIONS_PATTERNS_CONVENTIONS}} to reuse. Prefer reuse over new code.

THE PLAN MUST CONTAIN
1. CONTEXT — the problem, why now, the intended outcome
2. APPROACH — the recommended one only, not a survey of alternatives
3. FILES — specific paths to create or change. For repeated patterns, describe once with two or
   three representative paths; do not enumerate every file
4. REUSE — existing components to build on, with their paths
5. RISKS — what could go wrong, what is hard to reverse
6. VERIFICATION — how to prove it works end to end
7. OUT OF SCOPE — what a reader might expect and will not get

RULES
- Investigate before proposing. A plan written from assumptions is a guess with formatting.
- Recommend, do not enumerate. One approach, with the reasoning that selected it.
- Name the decision that would be expensive to reverse, and say so explicitly.
- If two readings of the request would produce materially different work, ask before planning.

STOP after the plan. Do not begin implementation.
```

### Parameters

| Placeholder | Guidance |
|---|---|
| `{{WHERE_TO_LOOK}}` | Concrete paths beat "the codebase" |
| `{{FUNCTIONS_PATTERNS_CONVENTIONS}}` | Naming the reuse target prevents parallel implementations |

### Failure signatures and exit criteria

| Signature | What it means | Response |
|---|---|---|
| Plan lists three options and no recommendation | Deferring the decision | Demand one approach and its reasoning |
| No file paths | Not investigated | Send it back; require reading first |
| Enumerates every file exhaustively | Padding | Ask for the pattern plus representatives |
| Fourth revision with no execution | Planning as procrastination | Set a revision cap and execute |
| Plan contradicts a settled decision | Context omitted the constraint | Add "already decided" to the preamble |

**Exit:** a reader who was not present could execute it. Test by asking what they would do first.

### Anti-patterns and when to switch

- **Planning without investigation** — plausible, wrong, and confident.
- **The plan that is really an essay.** If it has no file paths, it is not a plan.
- **Switch to [Supervisor](#7--supervisor)** when execution needs judgement per step rather than a
  fixed sequence.

### Cost, latency and model choice

Cheap relative to what it saves — one wrong implementation costs more than ten plans. Investigation
dominates the token spend. Use a strong model: this is the highest-leverage reasoning in the whole
workflow, and errors here propagate through everything downstream.

### Worked micro-example

```text
INVESTIGATE FIRST
Read src/auth/, tests/auth/, and any existing session-handling middleware.
Identify existing validation helpers to reuse.

THE PLAN MUST CONTAIN ...

Additional constraint for this task:
- Session storage was already decided (Redis, see ADR 0004). Do not re-open it.
- If the plan requires a schema migration, that is a hard-to-reverse decision: say so explicitly.

STOP after the plan.
```

---

## Orchestration patterns

## 5 — Sequential

**Use when** stages are known, ordered, and each has a checkable exit condition.

**Skip when** the order depends on findings — use [ReAct](#3--react).

### Prompt

```text
Execute these stages in order. Do not begin a stage until the previous stage's EXIT holds.

STAGE 1 — {{NAME}}
  INPUT:  {{WHAT_IT_RECEIVES}}
  DO:     {{WHAT_IT_PRODUCES}}
  EXIT:   {{CHECKABLE_CONDITION}}

STAGE 2 — {{NAME}}
  INPUT:  the artifact from stage 1
  DO:     {{...}}
  EXIT:   {{CHECKABLE_CONDITION}}

STAGE 3 — {{NAME}}
  ...

RULES
- Each stage hands over an ARTIFACT (a file, a diff, a report), not a description of one.
- If a stage's input is ambiguous, stop and name the ambiguity. Do not guess and continue —
  a guess at stage 1 becomes confident wrongness at stage 3.
- Do not skip ahead, even when a later stage looks trivial.
- Report per stage: exit condition met or not, and the artifact produced.
```

### Parameters

| Placeholder | Guidance |
|---|---|
| `{{CHECKABLE_CONDITION}}` | Externally verifiable. "Tests exist and fail for the intended reason" — not "stage complete" |
| Artifacts | Prefer files over messages: files survive the session |

### Failure signatures and exit criteria

| Signature | What it means | Response |
|---|---|---|
| Stage 3 confidently building on a stage-1 guess | Ambiguity laundered | Enforce stop-and-name |
| Stages blur together in one response | Not respecting exits | Separate calls per stage |
| A stage reports done with no artifact | Description mistaken for delivery | Require the artifact |
| Late stage exposes an early misunderstanding | Exit conditions too weak | Strengthen the early exits |

**Exit:** the final stage's condition holds and every intermediate artifact exists.

### Anti-patterns and when to switch

- **Stages with no exit conditions** — a list of instructions wearing a pipeline's clothes.
- **Parallelising a sequential chain** because it seems faster. If stage 2 needs stage 1's output,
  concurrency buys nothing and costs correctness.
- **Switch to [Supervisor](#7--supervisor)** when which stage runs next depends on results.

### Cost, latency and model choice

Predictable: roughly the sum of the stages, no re-reasoning overhead. Latency is the sum too —
this is the pattern's real cost. Stages can use *different* model tiers; the cheapest often works
for mechanical stages, with the strong model reserved for the stage where judgement lives.

### Worked micro-example

```text
STAGE 1 — SPECIFY
  DO:   Write the acceptance criteria for the feature as a numbered list.
  EXIT: Every criterion is objectively checkable. No criterion contains "correctly" or "properly".

STAGE 2 — TEST
  INPUT: the criteria from stage 1
  DO:    Write failing tests, one per criterion, named with its number.
  EXIT:  Tests run and fail for the intended reason — not on a typo or a missing import.

STAGE 3 — IMPLEMENT
  INPUT: the failing tests
  DO:    Minimum code to pass them.
  EXIT:  Full suite green; no test was modified to achieve it.
```

Stage 2's exit is the interesting one: "fails for the intended reason" catches the common case
where a test fails on a broken import and everyone declares red achieved.

---

## 6 — Parallel

**Use when** branches are genuinely independent, share no mutable state, and you have decided the
merge rule *before* fanning out.

**Skip when** branches touch the same files, or when you have not decided how to reconcile
disagreement. Both make parallelism a way to generate noise faster.

### Prompt

```text
BRANCHES — run independently. Each sees only its own instructions.

BRANCH A — {{PERSPECTIVE}}
  Examine {{SCOPE}} for {{CONCERN}}.
  Report: findings with locations and severity. Nothing outside your concern.

BRANCH B — {{PERSPECTIVE}}
  ...

BRANCH C — {{PERSPECTIVE}}
  ...

MERGE RULE — decided in advance, not after reading the results
- {{BRANCH_A}} findings are BLOCKING. Any finding stops the change.
- {{BRANCH_B}} findings are ADVISORY. Record; do not block.
- Conflict between branches resolves toward {{AUTHORITATIVE_BRANCH}}, because {{REASON}}.
- Duplicates across branches: keep the most specific statement, note the corroboration.

ISOLATION
No branch may read another's output. No branch may modify shared state.

OUTPUT
Merged report: blocking findings first, then advisory, then a note on anything two branches
disagreed about and how the merge rule resolved it.
```

### Parameters

| Placeholder | Guidance |
|---|---|
| `{{PERSPECTIVE}}` | Must be genuinely different. Three branches asking "is this good?" produce one answer three times |
| `{{AUTHORITATIVE_BRANCH}}` | Usually the safety or correctness branch. Decide before you see results |

### Failure signatures and exit criteria

| Signature | What it means | Response |
|---|---|---|
| Branches return near-identical findings | Perspectives insufficiently distinct | Collapse to one branch |
| Merge picks whatever was read last | No merge rule | Stop and write one |
| A branch comments outside its concern | Scope leaking | Restate "nothing outside your concern" |
| Merge conflicts in shared files | Branches were not independent | Serialise them |

**Exit:** all branches reported and the merge rule has been applied — including to disagreements.

### Anti-patterns and when to switch

- **Fan-out for the appearance of rigour.** Three reviewers with the same brief is one reviewer
  and triple the bill.
- **Merging by averaging.** A blocking security finding does not average with two approvals.
- **Switch to [Sequential](#5--sequential)** when a branch needs another's output. That is a
  dependency, and dependencies are sequential no matter how you draw the diagram.

### Cost, latency and model choice

Cost is the sum of branches — genuinely N× — while latency is the *max*. That is the whole trade:
you buy wall-clock with tokens. Only worth it when the branches are real. Branches can use
different tiers, and a merge step needs a strong model precisely because reconciling disagreement
is the hard part.

### Worked micro-example

```text
BRANCH A — SECURITY
  Examine the diff for credential handling, injection, and authorization gaps.
  Report findings with locations and severity. Nothing about style or performance.

BRANCH B — CORRECTNESS
  Examine the diff for logic errors, unhandled cases, and race conditions.
  Nothing about security or style.

BRANCH C — MAINTAINABILITY
  Examine the diff for duplication, unclear naming, and missing tests.
  Nothing about security or correctness.

MERGE RULE
- Branch A findings are BLOCKING.
- Branch B findings are BLOCKING if they can produce wrong output; otherwise advisory.
- Branch C findings are ADVISORY.
- A and B conflict: A wins. A vulnerability outranks an elegance argument.
```

---

## 7 — Supervisor

**Use when** work must be chosen, dispatched, judged and re-chosen — and the number of iterations
is not known up front.

**Skip when** the sequence is fixed. That is [Sequential](#5--sequential) with extra tokens.

### Prompt

```text
You coordinate. You do not do the work yourself unless it is trivial.

OBJECTIVE
{{OBJECTIVE}}
DONE WHEN: {{FALSIFIABLE_COMPLETION_CONDITION}}

EACH ITERATION
1. ASSESS   — run {{ASSESSMENT_STEP}}. State the current gap in concrete terms.
2. SELECT   — choose the highest-value next unit of work. Say why it, and not the alternatives.
3. DISPATCH — delegate with a complete brief: context, constraints, output contract, exit.
4. EVALUATE — check the result against its exit condition. Do not accept a self-report.
5. DECIDE   — iterate, escalate to a human, or stop.

BUDGETS
- At most {{N}} iterations.
- At most {{M}} units of work dispatched per iteration.
- Prefer small batches. A large batch that fails wastes the whole batch.

ESCALATE TO A HUMAN WHEN
- {{AMBIGUITY_THAT_WOULD_CHANGE_BEHAVIOUR}}
- The work needs {{ACCESS_YOU_DO_NOT_HAVE}}
- Two requirements conflict, or the work contradicts a settled decision
- The same unit has failed {{K}} times for different reasons — that is a design problem

NEVER
- Report progress you have not made. "Three done, two failing" is useful; "good progress" is not.
- Weaken an exit condition to make something pass.
- Continue past DONE WHEN to add improvements nobody asked for.
```

### Parameters

| Placeholder | Guidance |
|---|---|
| `{{ASSESSMENT_STEP}}` | Must be mechanical — a coverage report, a failing-test list. Not "consider how it's going" |
| `{{M}}` | Small. 2–5 units. Large batches fail expensively |
| `{{K}}` | 3. Three different failures on one unit means the unit is wrong, not the attempt |

### Failure signatures and exit criteria

| Signature | What it means | Response |
|---|---|---|
| Iterations continue past the objective | No enforced DONE WHEN | Make it mechanical |
| Work dispatched with thin briefs | Assuming shared context that does not exist | Require the full brief every time |
| Accepts sub-agent self-reports | No evaluation step | Verify against the exit condition independently |
| Fan-out grows each iteration | No budget | Enforce `{{M}}` |
| Same unit retried repeatedly | Should have escalated | Enforce the `{{K}}` rule |

**Exit:** the completion condition is mechanically true. Not "the supervisor believes it is."

### Anti-patterns and when to switch

- **The supervisor that does the work itself.** Coordination collapses into one long response and
  the evaluation step quietly disappears.
- **Delegation without a brief** — sub-agents start cold and re-derive context you already have.
- **Escalation as a dumping ground.** If everything escalates, the pattern is not earning its keep.
- **Switch to [Sequential](#5--sequential)** when you notice the iterations are always the same
  three steps in the same order.

### Cost, latency and model choice

The most expensive pattern: coordination overhead plus the work, plus re-assessment each round.
Budget it deliberately. The supervisor needs a strong model — selection and evaluation are the
hard parts — while dispatched units can often run a tier lower. If the supervisor is cheap and the
workers expensive, you have it backwards.

### Worked micro-example

```text
OBJECTIVE
Bring module X to full test coverage of its documented behaviours.
DONE WHEN: every documented behaviour has a passing test, and the suite is green.

EACH ITERATION
1. ASSESS   — run the coverage report; list behaviours with no test.
2. SELECT   — pick 3, preferring error paths over happy paths (they hide more defects).
3. DISPATCH — for each: the behaviour, its documented contract, the test file, "tests only,
              no implementation", exit = "test exists and fails for the intended reason".
4. EVALUATE — run the tests. Confirm each fails, and read the failure message.
5. DECIDE   — iterate or stop.

BUDGETS: 8 iterations, 3 units each.
ESCALATE WHEN: a behaviour's documented contract is ambiguous enough that two readings would
produce different tests.
```

---

## 8 — Human-on-the-loop

**Use when** the work touches anything irreversible, outward-facing, or requiring context the
system does not have.

**Skip when** everything is reversible and low-stakes. Gates on trivia train people to approve
reflexively, which disarms the gate that matters.

### Prompt

```text
Run autonomously within these boundaries. Surface decisions that are genuinely the human's.

PROCEED WITHOUT ASKING
- {{REVERSIBLE_ACTION_1}}
- {{REVERSIBLE_ACTION_2}}
- Anything you can undo without external effect

STOP AND ASK
- {{IRREVERSIBLE_ACTION}} — cannot be undone
- {{OUTWARD_FACING_ACTION}} — visible outside this system; may be cached or indexed
- {{AMBIGUOUS_DECISION}} — needs context you do not have
- Anything that spends money, sends a message, deletes data, or changes access

WHEN YOU STOP, PROVIDE
1. What you are about to do, in one sentence
2. Why it is needed
3. What it affects, and what happens if it is wrong
4. What you have already established — do not make the human re-derive it
5. The specific question, with options if there are discrete ones

DO NOT
- Ask for approval on things in the PROCEED list. Approval fatigue disarms the real gates.
- Proceed on a STOP item because it "seemed fine" or "was probably intended".
- Bundle an unapproved action with an approved one.
- Treat prior approval as standing authorisation for a similar later action.
```

### Parameters

| Placeholder | Guidance |
|---|---|
| `{{IRREVERSIBLE_ACTION}}` | Enumerate concretely: force-push, drop table, publish, send, rotate, revoke |
| `{{OUTWARD_FACING_ACTION}}` | Anything leaving the system. Publishing is not undone by deleting |
| `{{AMBIGUOUS_DECISION}}` | Where two readings produce materially different work |

### Failure signatures and exit criteria

| Signature | What it means | Response |
|---|---|---|
| Asks about everything | Boundaries too tight | Move reversible items to PROCEED |
| Proceeds on something irreversible | Boundaries too loose or unenumerated | Name the action type explicitly |
| Approval requests lack impact | Missing the "what it affects" field | Enforce all five fields |
| Human approves without reading | Fatigue — the pattern has failed | Reduce gates to the genuinely irreversible |
| Bundles unapproved with approved | Scope smuggling | Require separate approval per action |

**Exit:** the task completes, or it stops at a genuine decision point with a complete brief.

### Anti-patterns and when to switch

- **Human-in-the-way.** Approval on every step until approval is reflexive.
- **Approval as blame transfer** — a request framed so vaguely the human cannot evaluate it.
- **Standing authorisation.** "You approved deleting that branch" is not approval to delete this
  one.
- **Switch to full autonomy** when a class of decision has been approved identically many times
  and is genuinely reversible — then move it to PROCEED deliberately, not by drift.

### Cost, latency and model choice

Token cost is negligible; the cost is *human attention*, which is the scarcest resource in the
system. Optimise for fewer, better-framed interruptions. Latency is unbounded — the human may be
asleep — so anything on this path needs to tolerate an indefinite wait rather than time out into a
half-applied state.

### Worked micro-example

```text
PROCEED WITHOUT ASKING
- Reading any file; running tests; creating a branch; committing to that branch
- Editing files in src/ and tests/

STOP AND ASK
- Pushing to a remote — outward-facing
- Any git history rewrite — irreversible for anyone who has pulled
- Deleting or truncating anything under data/ — irreversible
- Changing dependency versions in a lockfile — affects reproducibility for everyone
- Any change to CI credentials or permissions

WHEN YOU STOP, PROVIDE all five fields.

DO NOT ask permission to run tests. You will be asked to stop doing that.
```

---

## Composition recipes

Patterns stack. Three combinations worth knowing, each with the reason it beats its parts.

### Recipe A — Specification pipeline

#### Planning → Reflection → Sequential → Human-on-the-loop

```text
Planning           produce a spec artifact
   ↓
Reflection         critique it: mechanical checks, then judgement
   ↓
Human-on-the-loop  approve the plan before any execution
   ↓
Sequential         test → implement → verify, with exits between
```

Use for work where being wrong is expensive and the wrongness is not obvious until late. The
reflection stage between planning and execution is what stops a confident misunderstanding
propagating into code. This is the loop this repository runs; see
[`../agentic-orchestration.md`](../agentic-orchestration.md).

### Recipe B — Multi-perspective review

#### Parallel → Reflection → Human-on-the-loop

```text
Parallel           N independent reviewers, distinct concerns, isolated
   ↓
Reflection         merge under the pre-declared rule; reconcile disagreements
   ↓
Human-on-the-loop  human decides on blocking findings only
```

Use when a change has several independent risk dimensions. The merge rule must exist before the
fan-out, or the reflection stage becomes an averaging machine.

### Recipe C — Diagnose then repair

#### ReAct → Planning → Sequential

```text
ReAct        narrow hypotheses to one confirmed cause
   ↓
Planning     design the fix — a different task with different failure modes
   ↓
Sequential   test that reproduces the bug → fix → verify
```

Use for incidents. The hard-won discipline is the boundary: do not let the diagnosis loop start
fixing. A ReAct loop that begins editing loses its ability to eliminate hypotheses, because it is
now changing the system it is measuring.

### Composition rules

1. **Compose deliberately.** Each layer multiplies cost and failure surface.
2. **One stop condition per layer**, and the outermost one wins.
3. **Do not nest supervisors.** Two levels of coordination means neither owns the budget.
4. **Human-on-the-loop belongs at the boundary**, not sprinkled through the middle.

---

## Portability notes

### Any LLM tool

Everything above is plain text. The only real requirements:

- **Long enough context** to hold the preamble, the module and the task.
- **Some way to act**, for tool use, ReAct, and anything that verifies its own work. Without tools,
  reflection and planning still work; the others degrade to roleplay.
- **A way to run the exit-condition check** that is not the model. If you must ask the model
  whether it is finished, expect to be told yes.

### Claude Code specifics

Quarantined here so the rest travels unchanged.

| Concept above | How it maps |
|---|---|
| Pattern module | A file in `.claude/commands/` — invoked as `/name`, with `$ARGUMENTS` |
| Persistent role + tool allowlist | A file in `.claude/agents/` with `tools:` frontmatter |
| Durable domain knowledge | A skill in `.claude/skills/<name>/SKILL.md`, loaded on demand |
| Human-on-the-loop gate | Plan mode: a written plan, approved before execution |
| Supervisor iteration | The `loop` skill for self-paced runs; scheduled routines for recurring ones |
| Externalised state | Files plus git. Task tracking for in-session progress |

Two things this environment gives you that a bare API does not: **tool allowlists per agent role**,
which is how least privilege stops being a comment; and **hooks**, which enforce gates the model
cannot talk its way past, because they run outside it.

### Adapting a module

1. Keep the shared preamble; it is the portable part.
2. Replace every `{{PLACEHOLDER}}` — an unfilled one reads as an instruction to invent.
3. Rewrite the failure signatures for your domain. The listed ones are generic; yours are specific
   and more useful.
4. Set budgets from measurement, not from feel.
5. Delete what you do not need. A module with three irrelevant sections gets skimmed, and skimmed
   prompts are ignored prompts.

---

## Quick reference

| Pattern | Shape | Exit when | Biggest failure |
|---|---|---|---|
| Reflection | produce → critique → revise | Mechanical checks pass, no blocking finding | Rubber-stamping |
| Tool use | constrained actions on real state | Task condition, or budget with a report | Tool sprawl |
| ReAct | thought → action → observation → update | One hypothesis survives and is confirmed | Thrashing |
| Planning | investigate → artifact → review | A stranger could execute it | Never converts to action |
| Sequential | stage → stage → stage | Final exit holds, all artifacts exist | Error propagation |
| Parallel | fan out → merge by rule | All branches in, rule applied | No merge rule |
| Supervisor | assess → select → dispatch → evaluate | Completion condition mechanically true | Unbounded fan-out |
| Human-on-the-loop | autonomous, escalate on boundary | Task done, or a complete brief at a decision | Approval fatigue |
