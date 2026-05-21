# <PROJECT-NAME> — <one-line description>

<2-3 sentence project summary: what it is, who it is for, and what makes this project different. Replace this paragraph.>

> **Agent operating model.** This file is a thin project spine for Claude Code. It is not a workflow engine. Agents must use concrete repo evidence, tests, and human gates instead of treating prompts as proof of correctness.

> **Provided by the `agentic-ai-features` plugin.** This file was authored by `/agentic-ai-features:init`. The skills and agents referenced below ship with that plugin; install it once globally (`/plugin install agentic-ai-features`) and the slash commands are available in every project that has a CLAUDE.md like this one.

## Start Here

- **First-time agent:** read this file, then the referenced docs that match your task.
- **Planning a feature:** `/agentic-ai-features:feature-planner` — produces a decision-complete plan before implementation.
- **Implementing one task:** `/agentic-ai-features:implement-task` — runs implementer + verifier + reviewer (each in its own agent context) for a single scoped task, then stops.
- **Walking a multi-task plan:** `/agentic-ai-features:task-loop` — autonomously walks a plan file task by task, commits between tasks, stops at human gates.
- **Auditing claimed work:** `/agentic-ai-features:check-completeness` — audits current repo evidence against acceptance criteria; includes a reachability pass and an AI-eval-coverage pass.

## Project Status

> Replace with the actual phase or milestone breakdown for this project. Keep this brief; details belong in docs.

- [ ] Discovery / project framing.
- [ ] Initial implementation plan.
- [ ] <Milestone 1>.
- [ ] <Milestone 2>.

## Hard Rules

> Replace placeholders with project-specific non-negotiables. Each rule must be one line and externally verifiable where possible.

**General**

- **One task at a time** — implementation work must have one task ID, one scope, and one acceptance checklist.
- **No silent scope expansion** — if implementation requires files or behavior outside the task scope, stop and ask.
- **Evidence over confidence** — do not mark work complete unless the listed verification commands or observations pass.
- **Fresh-agent isolation** — implementation, verification, and review must run in separate agent invocations. Never let one agent rubber-stamp its own work.
- **`/goal`-wrapped unattended runs** — when `/goal` is available (Claude Code ≥ v2.1.139), wrap any multi-turn unattended run (`/agentic-ai-features:task-loop`, long `/agentic-ai-features:check-completeness` sweeps) in `/goal` so an external evaluator — not the loop's own self-report — decides termination.

**AI-feature rules** (apply when the task touches prompts, model calls, or LLM output)

- **No silent model bumps** — the exact model string and tokenizer / API version are pinned in code or config; changing them needs a human gate.
- **Eval before merge** — every AI feature ships with an eval suite (golden outputs, regression set, or scored rubric). Every prompt edit re-runs it. No eval suite means no merge.
- **Cost / latency budget per feature** — each AI feature declares `$/call` and `p95 latency` budgets. Tests or monitoring must reject regressions over budget.
- **Prompt fingerprint in tests** — tests that exercise real prompts snapshot the exact prompt body so silent edits surface as test diffs.
- **No prompts in code without provenance** — every prompt string carries a comment or sibling file that names the eval set it was tuned against.

**Project-specific** (fill in)

- **<Rule 1>** — <one-line statement, with doc pointer if needed>.
- **<Rule 2>** — <one-line statement, with doc pointer if needed>.

## Human Gates

Agents must stop for human confirmation when any of these occurs:

- A new feature, public API, schema, dependency, permission, external service, deployment path, or data migration is introduced.
- The task scope is ambiguous or needs files not listed in the task.
- Verification requires subjective product judgment, visual approval, manual QA, credentials, paid services, or unavailable hardware.
- Tests or checks fail for a reason the agent cannot resolve without changing intent.
- A reviewer reports a risk that cannot be reduced to a concrete code or test change.
- An AI feature changes model, tokenizer, or API version (silent bumps forbidden by the hard rules).
- An AI feature's eval suite regresses below threshold or breaches a cost / latency budget.
- A prompt edit lands in a path that has no eval coverage yet.

## Common Commands

> Replace with the actual top-level commands for this project. Prefer commands that are safe for agents to run repeatedly.

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
  - [ ] <observable result>
- **Verification:**
  - `<command or concrete inspection>`
  - `<command or concrete inspection>`
```

**For AI-feature tasks**, add these blocks (mandatory whenever the task touches a prompt, model call, or LLM output):

```markdown
- **Eval criteria:**
  - Eval set: `<path/to/evals or harness name>`
  - Threshold: <pass condition, e.g. "≥ 0.85 on regression set, 0 hard fails on safety set">
- **Cost / latency budget:**
  - $/call: <upper bound>
  - p95 latency: <upper bound>
  - Tokens per call: <upper bound, optional>
- **Model pinning:**
  - Model: `<exact model string, e.g. claude-opus-4-7>`
  - Tokenizer / API version: `<version>`
```

Avoid vague acceptance criteria such as "clean", "robust", "done properly", "matches existing style", "good output", or "high quality" unless paired with concrete evidence or an eval threshold.

## Refs — Progressive Disclosure

Domain knowledge lives outside this file. Read only the refs whose trigger matches the current task.

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

Keep this file short. If a section needs more than a few lines of explanation, move the body into `docs/refs/` and link it from here.

## Skills + Agents (from the `agentic-ai-features` plugin)

| Slash command / Agent | Role |
|---|---|
| `/agentic-ai-features:feature-planner` | Creates a decision-complete feature or task plan. Does not write production code. |
| `/agentic-ai-features:implement-task` | Implements **one** scoped task end-to-end (implementer → verifier → reviewer), then stops. |
| `/agentic-ai-features:task-loop` | Walks a multi-task plan file autonomously. Same per-task pipeline as `implement-task`, plus commit-between-tasks and stop-at-human-gate. |
| `/agentic-ai-features:check-completeness` | Audits claimed work against acceptance criteria. Includes a reachability pass and an AI-eval-coverage pass. |
| `/agentic-ai-features:init` | (Re-)creates this CLAUDE.md in a project from the plugin's template. Idempotent. |
| `agentic-ai-features:task-implementer` (sub-agent) | Applies one task within explicit scope. Stops on scope expansion. |
| `agentic-ai-features:task-verifier` (sub-agent) | Checks measurable acceptance criteria. Marks subjective items as human verification required. |
| `agentic-ai-features:code-reviewer` (sub-agent) | Reviews risk, conventions, tests, docs, scope creep, and (for AI features) prompt-diff / eval / cost. Does not certify product correctness. |

## Recent Landmarks

> Rolling list of concise project-level events. Older history belongs in git and implementation notes.

- _(empty)_
