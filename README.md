# agentic-ai-features

A Claude Code plugin for **building AI features and AI apps agentically**: a foundation council that produces the project README before any code is written, a feature-planner, an autonomous task-loop orchestrator, three isolated implementer / verifier / reviewer sub-agents, a completeness auditor with reachability + AI-eval-coverage passes, and a project-spine template with AI-specific hard rules.

This is not a generic "do code with Claude" kit. It is opinionated about how AI work differs from CRUD work: prompts need eval suites, models must be pinned, cost / latency are first-class gates, "the tests pass" never equals "the feature works", and — before any of that — the team has to agree what they're building, for whom, and why.

## Why use it

Building AI features with an LLM-driven coding agent has a particular set of failure modes:

- **Code starts before the team agrees on the product** — vision, target user, architecture and risks live in the founder's head; the implementer guesses; features ship that nobody can confidently say are "good". The plugin's foundation council exists to surface that disagreement *before* the first task, not in code review.
- **Code is added but never wired** — the implementer writes the class, the tests pass against injected fakes, the build is green, and production never calls it. The user notices weeks later.
- **Prompts drift silently** — someone edits a system prompt; no test fails because no test snapshots the prompt body; output regresses for one of fifty user phrasings.
- **Models bump invisibly** — a SDK default updates, a feature flag flips, and suddenly the feature is on a different model with different cost / latency / behaviour.
- **Reviewers rubber-stamp themselves** — a single agent writes, verifies, and reviews its own output in one context window; it convinces itself the work is fine.

This plugin's whole shape is built around those failure modes.

## What it gives you

| Slash command | What it does |
|---|---|
| `/agentic-ai-features:init` | Drops a `CLAUDE.md` project spine into the current repo. Idempotent — refuses to overwrite. |
| `/agentic-ai-features:foundation` | Runs a 5-perspective council (user-advocate, investor-advocate, architect, security-auditor, skeptic) on a short pitch, then a synthesizer that produces `README.md`, `docs/foundation/PERSPECTIVES.md`, and a gating `docs/foundation/OPEN-DECISIONS.md`. **Mandatory before any feature work** — the three implementation skills below refuse to start until OPEN-DECISIONS is clean. |
| `/agentic-ai-features:feature-planner` | Produces a decision-complete plan (tasks with goal, scope, acceptance criteria, verification, AI-eval criteria, cost / latency budget) before any code is written. |
| `/agentic-ai-features:implement-task` | Implements **one** task. Dispatches implementer → verifier → reviewer as **three separate Agent invocations** so each step gets a fresh context window. Stops after one task. |
| `/agentic-ai-features:task-loop` | Walks a multi-task plan file autonomously. Same three-agent pipeline per task, plus commit between tasks, stop at human gates, journal to `docs/work-log.md` for cross-session recovery. |
| `/agentic-ai-features:check-completeness` | Audits claimed work against current repo evidence. Includes a **reachability pass** (catches "added but never called from `main()`") and an **AI-eval-coverage pass** (catches prompts without eval files). |

| Sub-agent | Role |
|---|---|
| `agentic-ai-features:user-advocate` | Foundation council seat — end-user adoption, UX friction, target-user clarity. |
| `agentic-ai-features:investor-advocate` | Foundation council seat — market, moat, defensibility, why-now, unit economics. |
| `agentic-ai-features:architect` | Foundation council seat — stack pillars, system boundaries, scaling ceilings, build/buy. |
| `agentic-ai-features:security-auditor` | Foundation council seat — threats, data flows, AI-safety (prompt injection, leak, misuse), compliance. |
| `agentic-ai-features:skeptic` | Foundation council seat — load-bearing assumptions, prior art, kill criteria. The brake on the council. |
| `agentic-ai-features:foundation-synthesizer` | Reads the five council drafts and produces `README.md` + `docs/foundation/`. Does not add a sixth opinion. |
| `agentic-ai-features:task-implementer` | Applies one task within explicit scope. Stops on scope expansion. |
| `agentic-ai-features:task-verifier` | Checks acceptance criteria against concrete evidence. Marks subjective items as `HUMAN` (not `PASS`). |
| `agentic-ai-features:code-reviewer` | Reviews scope, risk, tests, docs, and (for AI features) model pinning, prompt diff honesty, eval coverage, cost / latency budget. |

## Core design choices

1. **Foundation before features.** Before `feature-planner` will plan and before `task-loop` will walk, the project must have a `README.md` produced by the foundation council and a clean `OPEN-DECISIONS.md`. The plugin enforces this as a hard gate inside `feature-planner`, `implement-task`, and `task-loop`. The cost of two minutes' alignment up front is a fraction of the cost of three weeks of well-implemented wrong feature.

2. **Five seats at the foundation table, not one.** The council's perspectives — user-advocate, investor-advocate, architect, security-auditor, skeptic — run as **five independent Agent invocations in parallel**, each writing to its own file. A synthesizer reads all five and writes the README. No seat can override or silently dilute another; disagreement surfaces as an `OPEN-DECISIONS` item.

3. **Three results, not two.** Every verifier reports `PASS` / `FAIL` / `HUMAN`. `HUMAN` cannot be converted to `PASS` by another agent. Subjective product judgment (does the model's summary read well?) is not automatable; pretending it is is the single biggest source of silent regressions in AI feature work.

4. **Fresh-agent isolation, non-negotiable.** Implement, verify, and review each run in their own Agent invocation. The five council seats and the synthesizer follow the same rule. No "or apply the rules directly" escape hatch — that defeats the entire point of the multi-agent pipeline.

5. **Evidence over confidence.** A task is not done because the agent says so. It is done when the listed verification commands pass, the reviewer finds no blocker, and any `HUMAN` items have been cleared by the actual human.

6. **AI features get extra gates.** Model string pinning, prompt fingerprinting in tests, eval suite required for every prompt, declared cost / latency budgets. Each is a hard rule in the project spine the `init` skill drops in.

7. **Autonomy with brakes.** `task-loop` runs through long plans unattended, but stops hard at human gates and at the AI-feature-specific guardrails (eval regression, budget breach, model bump). It commits between tasks so the human always has a clean rollback point.

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

This repo doubles as a Claude Code marketplace and a plugin — add the marketplace once, then install the plugin from it.

```text
# 1. add this repo as a marketplace
/plugin marketplace add moinsen-dev/agentic-ai-features

# 2. install the plugin from it
/plugin install agentic-ai-features@moinsen-agentic-ai-features
```

After install, restart Claude Code (or run `/plugin` and pick "Reload") so the new slash commands and sub-agents register.

To pull updates later:

```text
/plugin marketplace update moinsen-agentic-ai-features
/plugin update agentic-ai-features@moinsen-agentic-ai-features
```

For local development, point the marketplace at your checkout instead of GitHub:

```text
/plugin marketplace add /absolute/path/to/agentic-ai-features
/plugin install agentic-ai-features@moinsen-agentic-ai-features
```

## Use

In a fresh project:

```
/agentic-ai-features:init
```

This drops `CLAUDE.md` into the cwd. Open it, fill in the project-specific placeholders (project description, hard rules, common commands, refs trigger map). The AI-feature hard rules and the `HUMAN`-gate triggers are pre-filled.

Then — **before any feature work** — run the foundation council:

```
/agentic-ai-features:foundation
# answer the 5–7 pitch questions; the skill then dispatches the five council seats
# (user-advocate, investor-advocate, architect, security-auditor, skeptic) in parallel,
# then the synthesizer. Outputs:
#   README.md                              ← project anchor
#   docs/foundation/PITCH.md               ← your pitch, verbatim
#   docs/foundation/perspectives/*.md      ← one file per seat
#   docs/foundation/PERSPECTIVES.md        ← consolidated protocol
#   docs/foundation/OPEN-DECISIONS.md      ← 3–7 gating decisions
```

Open `README.md` and `docs/foundation/OPEN-DECISIONS.md`. Resolve every decision (tick the box, write the call inline). While any item is unchecked, the implementation skills below refuse to start.

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
│   ├── plugin.json
│   └── marketplace.json
├── README.md
├── templates/
│   ├── CLAUDE.md           # dropped into projects by /agentic-ai-features:init
│   └── README.md           # skeleton filled by /agentic-ai-features:foundation
├── skills/
│   ├── init/SKILL.md
│   ├── foundation/SKILL.md
│   ├── feature-planner/SKILL.md
│   ├── implement-task/SKILL.md
│   ├── task-loop/SKILL.md
│   └── check-completeness/SKILL.md
└── agents/
    ├── user-advocate.md           # foundation council seat
    ├── investor-advocate.md       # foundation council seat
    ├── architect.md               # foundation council seat
    ├── security-auditor.md        # foundation council seat
    ├── skeptic.md                 # foundation council seat
    ├── foundation-synthesizer.md  # writes README + docs/foundation/
    ├── task-implementer.md
    ├── task-verifier.md
    └── code-reviewer.md
```

## Status

Version 0.3.0. Adds the foundation council (`/agentic-ai-features:foundation`, five perspective agents, synthesizer, README template, hard gate in `feature-planner` / `implement-task` / `task-loop`). The AI-feature gates and the foundation council are the parts most likely to grow with use (safety eval surface, batch-eval triggers, additional council seats per project type).

## License

MIT.
