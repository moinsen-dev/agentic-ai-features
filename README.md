# agentic-ai-features

A Claude Code plugin for **building AI features and AI apps agentically**: a feature-planner, an autonomous task-loop orchestrator, three isolated implementer / verifier / reviewer sub-agents, a completeness auditor with reachability + AI-eval-coverage passes, and a project-spine template with AI-specific hard rules.

This is not a generic "do code with Claude" kit. It is opinionated about how AI work differs from CRUD work: prompts need eval suites, models must be pinned, cost / latency are first-class gates, and "the tests pass" never equals "the feature works".

## Why use it

Building AI features with an LLM-driven coding agent has a particular set of failure modes:

- **Code is added but never wired** — the implementer writes the class, the tests pass against injected fakes, the build is green, and production never calls it. The user notices weeks later.
- **Prompts drift silently** — someone edits a system prompt; no test fails because no test snapshots the prompt body; output regresses for one of fifty user phrasings.
- **Models bump invisibly** — a SDK default updates, a feature flag flips, and suddenly the feature is on a different model with different cost / latency / behaviour.
- **Reviewers rubber-stamp themselves** — a single agent writes, verifies, and reviews its own output in one context window; it convinces itself the work is fine.

This plugin's whole shape is built around those failure modes.

## What it gives you

| Slash command | What it does |
|---|---|
| `/agentic-ai-features:init` | Drops a `CLAUDE.md` project spine into the current repo. Idempotent — refuses to overwrite. |
| `/agentic-ai-features:feature-planner` | Produces a decision-complete plan (tasks with goal, scope, acceptance criteria, verification, AI-eval criteria, cost / latency budget) before any code is written. |
| `/agentic-ai-features:implement-task` | Implements **one** task. Dispatches implementer → verifier → reviewer as **three separate Agent invocations** so each step gets a fresh context window. Stops after one task. |
| `/agentic-ai-features:task-loop` | Walks a multi-task plan file autonomously. Same three-agent pipeline per task, plus commit between tasks, stop at human gates, journal to `docs/work-log.md` for cross-session recovery. |
| `/agentic-ai-features:check-completeness` | Audits claimed work against current repo evidence. Includes a **reachability pass** (catches "added but never called from `main()`") and an **AI-eval-coverage pass** (catches prompts without eval files). |

| Sub-agent | Role |
|---|---|
| `agentic-ai-features:task-implementer` | Applies one task within explicit scope. Stops on scope expansion. |
| `agentic-ai-features:task-verifier` | Checks acceptance criteria against concrete evidence. Marks subjective items as `HUMAN` (not `PASS`). |
| `agentic-ai-features:code-reviewer` | Reviews scope, risk, tests, docs, and (for AI features) model pinning, prompt diff honesty, eval coverage, cost / latency budget. |

## Core design choices

1. **Three results, not two.** Every verifier reports `PASS` / `FAIL` / `HUMAN`. `HUMAN` cannot be converted to `PASS` by another agent. Subjective product judgment (does the model's summary read well?) is not automatable; pretending it is is the single biggest source of silent regressions in AI feature work.

2. **Fresh-agent isolation, non-negotiable.** Implement, verify, and review each run in their own Agent invocation. No "or apply the rules directly" escape hatch — that defeats the entire point of the multi-agent pipeline.

3. **Evidence over confidence.** A task is not done because the agent says so. It is done when the listed verification commands pass, the reviewer finds no blocker, and any `HUMAN` items have been cleared by the actual human.

4. **AI features get extra gates.** Model string pinning, prompt fingerprinting in tests, eval suite required for every prompt, declared cost / latency budgets. Each is a hard rule in the project spine the `init` skill drops in.

5. **Autonomy with brakes.** `task-loop` runs through long plans unattended, but stops hard at human gates and at the AI-feature-specific guardrails (eval regression, budget breach, model bump). It commits between tasks so the human always has a clean rollback point.

## Pairing with `/goal`

`/goal` (Claude Code ≥ v2.1.139) is the recommended way to drive `task-loop` unattended. It wraps the session with an external evaluator that, after every turn, checks whether the loop's stated end-state actually holds — independent of what the loop itself claims. The loop's internal stop conditions are still useful (they make the next-step decision); `/goal` is the second pair of eyes.

The project spine's hard rules treat this as the default for any multi-turn unattended run.

A typical pattern:

```text
/goal Plan walk complete — every task in docs/plans/<feature>.md has a green
commit from /agentic-ai-features:task-loop (Verifier PASS, Reviewer approved
or approved-with-notes), OR a human gate is surfaced and awaiting input,
OR the loop reports a hard stop. Stop after at most 50 turns.

/agentic-ai-features:task-loop --plan docs/plans/<feature>.md
```

Why bother?

- Self-stop is unreliable. A loop can drift into "almost done, one more try" forever.
- `/goal`'s evaluator sees the whole transcript, so it catches "loop has been on TASK-017 for 8 turns" patterns the per-task retry counter can't.
- Cost / token clauses can go straight into the goal condition.
- On `--resume`, both the loop's journal (`docs/work-log.md`) and the goal condition restore — they compose cleanly.

`/goal` is not a replacement for the loop. The loop is the workflow (plan walking, three-agent pipeline, commit between tasks, journal). `/goal` is the termination check on top.

If `/goal` is unavailable (older Claude Code, or hooks disabled), the loop still runs in self-stop mode — but the journal entry should note that the external gate was absent so audits can flag it.

## Install

```bash
# install from local checkout while developing
claude --plugin-dir /path/to/agentic-ai-features

# once published to a marketplace
/plugin install agentic-ai-features
```

## Use

In a fresh project:

```
/agentic-ai-features:init
```

This drops `CLAUDE.md` into the cwd. Open it, fill in the project-specific placeholders (project description, hard rules, common commands, refs trigger map). The AI-feature hard rules and the `HUMAN`-gate triggers are pre-filled.

Then, for each feature:

```
/agentic-ai-features:feature-planner
# answer the planner's questions; it produces a plan file
/agentic-ai-features:task-loop --plan docs/plans/<feature>.md
# walks the plan, stops at human gates
```

For a single ad-hoc task (no plan file):

```
/agentic-ai-features:implement-task
```

Before shipping:

```
/agentic-ai-features:check-completeness reachability
# verifies every new lifecycle method and coordinator class is actually reached from a production entry point
/agentic-ai-features:check-completeness ai-evals
# verifies every prompt has an eval file
```

## Repository layout

```
agentic-ai-features/
├── .claude-plugin/
│   └── plugin.json
├── README.md
├── templates/
│   └── CLAUDE.md           # dropped into projects by /agentic-ai-features:init
├── skills/
│   ├── init/SKILL.md
│   ├── feature-planner/SKILL.md
│   ├── implement-task/SKILL.md
│   ├── task-loop/SKILL.md
│   └── check-completeness/SKILL.md
└── agents/
    ├── task-implementer.md
    ├── task-verifier.md
    └── code-reviewer.md
```

## Status

Version 0.1.0. The shape is stable; the AI-feature gates are the part most likely to grow with use (safety eval surface, batch-eval triggers, prompt-fingerprint formats).

## License

MIT.
