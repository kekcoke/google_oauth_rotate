Good question, and the answer is: the split applies, but the boundary you've drawn is in the wrong place — and one part of it is actively backwards for this project. Let me ground the numbers first.

  Ran 1 shell command

Yes, it applies — but not at the boundary you drew, and half of it is inverted.

The reframe

"Opus plans, Sonnet implements" is the right instinct with the wrong seam. The question that actually predicts whether a cheaper model is safe on a task is:

▎ If this comes out wrong, does a machine tell you?

Where there's a red/green oracle — a failing test, an eslint rule, a CI gate — a cheaper model is safe because the loop self-corrects. Where correctness is a judgment call with no automated signal, model strength is the only thing standing between you and a confident wrong answer that ships.

That reframing moves the line, because this repo has an unusual property: the specs already did most of the thinking. 114 MUSTs written as observable, falsifiable behavior; 138 test IDs each mapped to exactly one requirement; 32 named fixtures. Implementation here is far more determined than in a typical codebase. That's precisely the condition under which a smaller model succeeds.

Where your split is backwards

You said Sonnet for "implementation and validation testing." The testing half is the risky one, and here's why:

In TDD the test is the oracle. A wrong implementation fails a good test — caught. A wrong test passes — silent, and now it's load-bearing. The asymmetry is total.

Concrete case from this repo. T-002-06 proves ten concurrent getValidToken calls cause exactly one token exchange. Write it without a call-count assertion and it passes on day one, passes forever, and proves nothing — while single-flight is quietly broken. Your own test-author agent already warns about this: "Assert call counts wherever 'exactly once' is the requirement — single-flight, no-retry on invalid_grant, one alert not N. These pass accidentally without a count assertion."

Omitting a count assertion is exactly the kind of thing a weaker model does. So:

Sonnet on green: safe. Sonnet on red: that's where you lose.

There's a second wrinkle I just measured. mac's test plan elaborates only its 19 own spec-200 rows with full Given/When/Then/Fixture. Another 33 inherited rows carry level and fixture only, and the remaining ~86 test IDs are covered by range notation (T-002-01 … T-002-19) with the "what it proves" statement living back in the shared spec. So test authoring splits cleanly:

┌────────────────┬──────────────────────────┬────────────────────────────┐
│      Rows      │  Specification quality   │           Model            │
├────────────────┼──────────────────────────┼────────────────────────────┤
│ 19 elaborated  │ Given/When/Then/Fixture  │ Sonnet — it's              │
│ spec-200 rows  │ spelled out              │ transcription              │
├────────────────┼──────────────────────────┼────────────────────────────┤
│ ~119 inherited │ Must read the shared     │ Opus, especially the       │
│  /             │ spec and interpret for   │ time/concurrency/negative  │
│ range-covered  │ mac                      │ ones                       │
└────────────────┴──────────────────────────┴────────────────────────────┘

Model assignment by step

Step: 1 Settle specs
Work: Promote 10 specs draft→accepted, resolve B1–B12, write ADR 0008
Model: Opus
Why: A ratchet — after this each change costs an ADR. Tiny token volume,
maximum consequence, zero
────────────────────────────────────────
Step: 2 Scaffold
Work: package.json, Dockerfile, compose, eslint, .c8rc
Model: Sonnet
Why: Every decision is already made in IMPLEMENTATION.md (deps, base image,
the
three lint rules). Pure transcription with docker build as the oracle
─────────────────────────
Step: 2b Config table + open question
Work: B8 defaults; the REd-close false alarm
Model: Opus
Why: An unresolved design question that may need a superseding requirement     ─────────────────────────
Step: 3 Walking skeleton
Work: ~30 test IDs, B2 consent container, first real consent
Model: Opus
Why: The integration nobody has done: Docker networking + SQLite + volume
semantics + real OAuth. Failure mode is silent — a record written to the
wrong store looks identical to one never written
────────────────────────────────────────                                       Step: 4–6 Backfill
Work: ~108 remaining test IDs
Model: Split per phase — Opus writes red for time/concurrency/negative cases,
Sonnet writes red for elaborated rows, Sonnet green throughout
Why: Green has a machine oracle. Red doesn't
────────────────────────────────────────
Step: Any Security review
Work: token-security-reviewer
Model: Opus, always
Why: It hunts for absence — no token in a log, no credential in a layer.
Proving absence is strictly harder than proving presence
────────────────────────────────────────
Step: Any /verify, CI triage
Work: Run gates, report
Model: Sonnet
Why: Mechanical. Escalate on a failure you can't explain in one line

Rough shape: Opus on steps 1, 2b, 3 and the red phases of 4–6; Sonnet on step 2 and all green work — which is where the bulk of the ~108 remaining test IDs live.

Not worth using Haiku here. The coordination overhead exceeds the savings at this task size.

Session structure

One spec per session, following the topological order already in the plan: {001,009} → {004,007,008} → 002 → 003 → 005 → 006 → 200.

The full mac context surface is 1,916 lines (IMPLEMENTATION.md 462, test-plan  159, spec-200 130, nine sly 27k tokens if you loadit all. That's fine once. It's ruinous re-derived across twenty sessions.

The fix is already in the repo: mac/IMPLEMENTATION.md is the compression       artifact. Its Standing deno session re-derivesB1–B12. That's ~462 lines replacing what took two Explore agents and ~140k subagent tokens to establish the first time.                                  
A session should load: IMPLEMENTATION.md + the one target spec + the matching test-plan rows. Not the other eight specs.

Prompt shape

Read mac/IMPLEMENTATION.md — Standing decisions are binding.

Step 4, spec-001 only. Write failing tests for T-001-04, T-001-05,             T-001-06 (batch of 3, not
                                                                               Read: specs/spec-001-tokend
mac/specs/test-plan.md (inherited coverage rows + fixture catalogue).          Do not read the other spe
Confirm each test fails for the intended reason before stopping.               Assert call counts on anyy once".

Four properties that make it cheap: names the one step, names specific test IDs, names which files to read and which not to, and states a checkable stop   condition. What it does nt — that's the agent's jobto fetch, and pasting doubles the cost.

Token levers specific to this repo

/option-status --option mac is your state oracle. Reloading "where am I" by re-reading specs costs ~27k tokens. One command that reports requirement→test→implementation coverage costs a fraction of that. Start every backfill session with it.

Fixtures are built once. All 32 land in step 3. Steps 4–6 reference them by name (clock-jump, google-invalid-grant, store-unreachable) instead of re-describing the setup. That's the single biggest saving across the long tail.
                                                                               Batch 2–5 test IDs, neverready encodes this. A 19-test batch means a lonith all the output incontext.

Filter the test run. node --test --test-name-pattern 'T-001-0[456]' while      iterating; full suite onlst output is the quiettoken sink in TDD work.

Let CI do verification you'd otherwise redo locally. It already runs both traceability gates, markdownlint, links and the secret scans.

Subagents for exploration, not for building. A subagent starts cold and re-derives context — expensive. But it keeps tool output out of the parent, which is right for "find every place X appears." Wrong for the build loop, where you need the state.

What would change this advice                                                 
If the specs were vaguer, I'd push Opus much further down the stack — the whole recommendation rests on 114 MUSTs being falsifiable and 138 test IDs     being pre-mapped. And if ore than one split (theplan already says to split at the consent boundary, never the Docker           boundary), that's a signar than specified, and steps 4–6 deserve more Opus tha

One structural safeguard worth keeping regardless of model: the two eslint rules from step 2 that balowlist and process.envoutside lib/config.js. Those convert the project's two most-violated invariants from judgment into a machine oracle — which is exactly what makes a cheaper model safe on everything downstream of them.


