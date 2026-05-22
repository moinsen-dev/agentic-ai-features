# <PROJECT-NAME> — <one-line description>

<2-3 sentence project summary: what it is, who it is for, and what makes this project different. Replace this paragraph.>

> **Agent operating model.** This file is the Codex project spine for `agentic-ai-features`. Agents must use concrete repo evidence, tests, and human gates instead of treating prompts as proof of correctness.

> **Provided by the `agentic-ai-features` plugin.** In Codex, the plugin exposes skills through `.codex-plugin/plugin.json`. In Claude Code, the same workflows are available through `/agentic-ai-features:*` slash commands.

## Start Here

- **First-time agent:** read this file, then `README.md`, then any `docs/refs/` triggers that match your task.
- **Foundation first:** run the `foundation` skill. It produces `README.md`, `docs/foundation/`, hydrates this spine, and creates `docs/refs/`. This is mandatory before feature work.
- **Planning a feature:** run the `feature-planner` skill. It produces a decision-complete plan before implementation.
- **Implementing one task:** run the `implement-task` skill only for a single scoped task.
- **Walking a plan:** run the `task-loop` skill for multi-task plans, stopping at human gates.
- **Auditing claimed work:** run the `check-completeness` skill before handoff or release gates.

## Foundation Required

Before `feature-planner`, `implement-task`, or `task-loop` starts, the following must exist:

- `README.md` — project anchor produced by the foundation council.
- `docs/foundation/OPEN-DECISIONS.md` — gating checklist with every item ticked.
- `docs/refs/` — progressive-disclosure references linked from this spine.

While any item in `OPEN-DECISIONS.md` is unchecked (`- [ ]`), implementation work must stop and ask the human to resolve the decision inline.

## Project Status

- [ ] Discovery / project framing.
- [ ] Initial implementation plan.
- [ ] <Milestone 1>.
- [ ] <Milestone 2>.

## Hard Rules

**General**

- **Foundation gate** — no implementation skill may start without `README.md` present and a clean `docs/foundation/OPEN-DECISIONS.md`.
- **One task at a time** — implementation work must have one task ID, one scope, and one acceptance checklist.
- **No silent scope expansion** — if implementation requires files or behavior outside the task scope, stop and ask.
- **Evidence over confidence** — do not mark work complete unless listed verification commands or observations pass.
- **Fresh-agent isolation** — implementation, verification, and review should run in separate agent contexts when the platform allows it. If Codex sub-agents are unavailable or not authorized, surface the limitation as a human gate instead of self-reviewing.
- **No raw self-certification** — another agent's summary is not evidence. Verify with repo state, command output, or explicit human approval.

**AI-feature rules** apply when the task touches prompts, model calls, or LLM output:

- **No silent model bumps** — exact model string and tokenizer/API version are pinned in code or config; changing them needs a human gate.
- **Eval before merge** — every AI feature ships with an eval suite. Every prompt edit re-runs it. No eval suite means no merge.
- **Cost / latency budget per feature** — each AI feature declares `$/call` and `p95 latency` budgets.
- **Prompt fingerprint in tests** — tests that exercise real prompts snapshot the exact prompt body so silent edits surface as test diffs.
- **No prompts in code without provenance** — every prompt string names the eval set it was tuned against in a comment or sibling file.

**Project-specific**

- **<Rule 1>** — <one-line statement, with doc pointer if needed>.
- **<Rule 2>** — <one-line statement, with doc pointer if needed>.

## Human Gates

Agents must stop for human confirmation when any of these occurs:

- A new feature, public API, schema, dependency, permission, external service, deployment path, or data migration is introduced.
- The task scope is ambiguous or needs files not listed in the task.
- Verification requires subjective product judgment, visual approval, credentials, paid services, unavailable hardware, or unavailable platform sub-agents.
- Tests or checks fail for a reason the agent cannot resolve without changing intent.
- A reviewer reports a risk that cannot be reduced to a concrete code or test change.
- An AI feature changes model, tokenizer, or API version.
- An AI feature's eval suite regresses below threshold or breaches a cost / latency budget.
- A prompt edit lands in a path that has no eval coverage yet.

## Common Commands

| Command | Purpose | Required before done? |
|---|---|---|
| `<cmd>` | <what it checks or builds> | Yes / No |
| `<cmd>` | <what it checks or builds> | Yes / No |

## Task Format

Use this format for implementation tasks. Keep tasks small enough that one agent can hold the relevant files in context.

```markdown
### TASK-001: <imperative title>

- **Goal:** <user-visible or repo-visible outcome>
- **Scope:** <files, modules, or behavior allowed to change>
- **Out of scope:** <explicitly excluded behavior>
- **Depends on:** <task IDs or "none">
- **Human gate:** <none, or exact approval needed before implementation>
- **Acceptance criteria:**
  - [ ] <observable result>
- **Verification:**
  - `<command or concrete inspection>`
```

For AI-feature tasks, add:

```markdown
- **Eval criteria:**
  - Eval set: `<path/to/evals or harness name>`
  - Threshold: <pass condition, e.g. ">= 0.85 on regression set, 0 hard fails on safety set">
- **Cost / latency budget:**
  - $/call: <upper bound>
  - p95 latency: <upper bound>
- **Model pinning:**
  - Model: `<exact model string>`
  - Tokenizer / API version: `<version>`
```

## Refs — Progressive Disclosure

| If you are... | Read |
|---|---|
| <touching path X or behavior Y> | `docs/refs/<topic>.md` |
| <touching path X or behavior Y> | `docs/refs/<topic>.md` |

## Self-Improving Rule

When a task reveals durable project knowledge, capture it where future agents can find it:

- New gotcha or domain rule -> add it to the relevant `docs/refs/<topic>.md`.
- New top-level command or hard rule -> update this file.
- New milestone status -> update **Project Status**.
- New ref topic -> create `docs/refs/<topic>.md` and add a trigger row above.

## Skills + Agents

| Skill or role brief | Role |
|---|---|
| `foundation` | Runs the 5-perspective foundation council and produces `README.md` plus `docs/foundation/`. |
| `feature-planner` | Creates a decision-complete feature or task plan. |
| `implement-task` | Implements one scoped task through implementer, verifier, and reviewer roles when isolation is available. |
| `task-loop` | Walks a multi-task plan file and stops at human gates. |
| `check-completeness` | Audits claimed work against acceptance criteria, reachability, and AI eval coverage. |
| `agents/user-advocate.md` | Foundation perspective: end-user adoption, UX friction, target-user clarity. |
| `agents/investor-advocate.md` | Foundation perspective: market, moat, defensibility, why-now, unit economics. |
| `agents/architect.md` | Foundation perspective: stack pillars, system boundaries, scaling ceilings, build/buy. |
| `agents/security-auditor.md` | Foundation perspective: threats, data flows, AI safety, compliance. |
| `agents/skeptic.md` | Foundation perspective: load-bearing assumptions, prior art, kill criteria. |
| `agents/foundation-spine-hydrator.md` | Post-foundation role: hydrates `AGENTS.md` and `docs/refs/` from foundation output. |
| `agents/task-implementer.md` | Applies one task within explicit scope. |
| `agents/task-verifier.md` | Checks measurable acceptance criteria and reports `PASS`, `FAIL`, or `HUMAN`. |
| `agents/code-reviewer.md` | Reviews risk, conventions, tests, docs, scope creep, prompt diffs, evals, and cost. |

## Recent Landmarks

- _(empty)_
