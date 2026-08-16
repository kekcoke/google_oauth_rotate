# Prompt Library

Portable, project-independent prompt scaffolding for the eight agentic patterns this repository
runs on. Nothing here depends on OAuth, on Node, or on Claude Code — copy a module into any LLM
tool and fill in the parameters.

| File | What it is |
|---|---|
| [`agentic-pattern-library.md`](agentic-pattern-library.md) | The composer preamble, a selection heuristic, and eight self-contained pattern modules |

## How to use it

1. **Pick the pattern** from the selection heuristic at the top of the library. Most tasks need
   one; some need a composition, and the library ends with three tested recipes.
2. **Copy the shared preamble**, filling in role, context, constraints, output contract and — the
   field people skip and then regret — the **stop condition**.
3. **Copy the pattern module** underneath it.
4. **Read the failure signatures** for that pattern before you run it, so you recognise the
   failure while it is happening rather than after.

Each module carries its prompt block, parameters, failure signatures and exit criteria,
anti-patterns with the signal to switch, a cost and model-tier note, and a filled-in micro-example
so the template is never read cold.

## The eight patterns

**Core workflow** — how one agent behaves:

| Pattern | One line |
|---|---|
| Reflection | Produce, critique against stated criteria, revise |
| Tool use | Act through a constrained toolset rather than emitting text and hoping |
| ReAct | Choose each action after seeing the previous observation |
| Planning | Decompose into a durable artifact, then execute it |

**Multi-agent and orchestration** — how several agents or stages relate:

| Pattern | One line |
|---|---|
| Sequential | Stage N's output is stage N+1's input, with exit conditions between |
| Parallel | Independent branches run concurrently, then merge under explicit criteria |
| Supervisor | A coordinator decomposes, dispatches, evaluates, and decides whether to iterate |
| Human-on-the-loop | Runs autonomously; surfaces the decisions that are genuinely a human's |

## Seeing them in context

[`../agentic-orchestration.md`](../agentic-orchestration.md) maps all eight onto this
repository's actual agents, commands and CI — including an honest note on which two are only
partially built, and a worked trace of a real multi-pattern run that went wrong in an instructive
way.

## The three rules that outrank any template

1. **A falsifiable exit condition.** Checkable by something that is not the model. Without one, a
   loop stops arbitrarily or not at all.
2. **Externalised state.** If the loop's memory is the conversation, it dies with the context
   window and cannot be resumed or audited.
3. **A gate the model cannot talk past.** Tests, a linter, CI, a human. Self-assessment is
   generous.

A weak prompt with these three is more useful than an elegant one without them.
