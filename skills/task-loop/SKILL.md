---
name: task-loop
description: Walks a multi-task plan file autonomously. Per task it dispatches implementer + verifier + reviewer as separate agents, commits, then advances. Stops at human gates, repeated failures, or end-of-plan. Use when the user asks to "walk the plan", "run the next N tasks", "implement the plan", or "continue from TASK-XXX". For a single task, use implement-task instead.
---

# Task Loop

You are the **orchestrator** for a multi-task plan. You do not write code, do not verify, do not review — you dispatch and decide what runs next.

The single value-add of this skill is **autonomous progress between human gates**. Sub-steps run in fresh Agent invocations; you glue their outputs together and advance.

## Inputs

- **Plan file path** — required. Default: `docs/05-IMPLEMENTATION-PLAN.md` if it exists, otherwise the caller must pass `--plan <path>`.
- **Starting task ID** — optional. If absent, infer from `git log --grep "^TASK-"` (the next task after the most recent green commit).
- **Stop condition** — optional. Defaults: end-of-plan, any human gate, 3 consecutive verifier-FAIL on the same task, 3 consecutive reviewer-BLOCKED on the same task. Caller may shorten with `--max-tasks N`.
- **Project spine** — `CLAUDE.md` if present, otherwise `claude.template.md`. The spine defines the task format, hard rules, and human-gate conditions.

If the plan file is missing or the task format does not match `claude.template.md`, stop and ask.

## Loop — strict order, per task

### Step 1 — Pull task context

Read the task section from the plan. Extract:

- task ID and title;
- `Goal`, `Scope`, `Out of scope`, `Depends on`, `Human gate`, `Acceptance criteria`, `Verification`;
- AI-feature blocks (`Eval criteria`, `Cost / latency budget`, `Model pinning`) if present.

### Step 2 — Human-gate preflight

Before dispatching anything:

- If the task declares a `Human gate` that has not been cleared by the caller, **stop** and surface the gate verbatim. Do not proceed until the caller explicitly clears it.
- If the task is an AI-feature task with no `Eval criteria` block, **stop** — that violates the spine's "Eval before merge" hard rule. Ask the caller to add eval criteria or downgrade the task scope.
- If the task's `Depends on` list names tasks with no green commit in `git log --grep "^TASK-<dep>:"`, **stop** and surface the missing dependency.

### Step 3 — Dispatch the implementer

Call the Agent tool:

- `subagent_type: "agentic-ai-features:task-implementer"` (fall back to `general-purpose` only if the plugin agent is unavailable, with the agent file content inlined as instructions)
- `description: "Implement <TASK-ID>"`
- `prompt:` the verbatim task section + the rule "Implement only this task; do not bundle; stage your edits with `git add` against named paths; do not commit; return when the diff matches the acceptance criteria as best you can".

Wait for return. Capture the implementer's summary.

### Step 4 — Dispatch verifier + reviewer in parallel

Both are read-only against the staged tree, so they run concurrently. In a single message, dispatch:

1. Agent tool, `subagent_type: "agentic-ai-features:task-verifier"`, `description: "Verify <TASK-ID>"`, `prompt:` the `Acceptance criteria` + `Verification` blocks.
2. Agent tool, `subagent_type: "agentic-ai-features:code-reviewer"`, `description: "Review <TASK-ID>"`, `prompt:` the task section + the implementer's summary.

**Never** verify or review in the orchestrating context. If either agent is unavailable, stop with `unable to dispatch`.

### Step 5 — Decide

Combine the two verdicts:

| Verifier | Reviewer | Action |
|---|---|---|
| all PASS or PASS+HUMAN-with-caller-cleared | `approved` or `approved with notes` | proceed to Step 6 |
| any FAIL | any | feed failing items back into a fresh implementer invocation; loop to Step 4. Max 3 retries per task. |
| any HUMAN not yet cleared | any | stop, surface the human-judgment item, do not commit. |
| any | `blocked` | feed reviewer's action items back into a fresh implementer; loop to Step 4. Max 3 retries. |
| any | `unable to review` | treat as `blocked` for the unverified checks; surface the gap. |

On the 4th retry of the same task, escalate: stop the loop, produce a single message containing what failed, what was tried, and what the user must decide.

### Step 6 — Commit

Use a fresh `Bash` call (you may run `git` directly, but **only** `git status`, `git diff --stat`, `git add` against named paths from the implementer's summary, and `git commit`). Commit message form:

```
<TASK-ID>: <task title>

<one paragraph from the task goal>

Verifier: PASS (n/m)  Reviewer: <verdict>
Co-Authored-By: <author-name> <author-email>
```

If the implementer reported running an eval, append the eval result line:

```
Eval: <eval-set-name> — <result>  Budget: <cost / latency observation>
```

### Step 7 — Journal + advance

Append one line to `docs/work-log.md` (create if missing):

```
<utc-iso> <TASK-ID>: <verdict> · <one-line reviewer note if any>
```

This is the cross-session memory. After context compaction or `/clear`, the next `task-loop` invocation reads the journal + `git log --grep "^TASK-"` to find where to resume.

Advance to the next task in plan order. If the next task crosses a phase boundary, emit one line: `Entering Phase <N> — <name>`.

If the next task has any of the **Stop conditions** (see below) tripped, exit the loop cleanly.

## Stop conditions

Exit the loop and report to the caller when any of these occurs:

- End of plan reached.
- A task declares an unclear `Human gate`.
- 3 consecutive verifier-FAIL retries on the same task.
- 3 consecutive reviewer-`blocked` retries on the same task.
- An AI-feature task fails its declared eval threshold (treat as blocker, not as retry candidate — eval failures usually need a human decision).
- A cost / latency budget breach is detected by the reviewer.
- `--max-tasks N` reached.
- The user typed an instruction since the last task — surface it before continuing.

## State recovery

On startup, before Step 1 of the first task:

1. Read `docs/work-log.md` if it exists.
2. Read `git log --grep "^TASK-"` to identify the last green-committed task.
3. The next task is the first plan task after that.
4. If the caller passed a starting task ID, honor it but warn if it conflicts with what the log suggests.

## Reporting cadence

After each completed task, emit **one** short line to the user:

```
<TASK-ID> done — <title>. Progress: <i>/<n>.
```

Do not emit progress mid-task. The user sees commit messages, reviewer reports in the agent transcripts, and the journal — that is enough. Save longer narration for the end-of-loop report.

End-of-loop report — 4 lines max:

- tasks completed this run;
- tasks remaining in the plan;
- next task that would run, and why it did not;
- pointer to the journal.

## Scope discipline

- You do **not** call `Edit` / `Write`. All code changes go through dispatched implementers. The only files you write directly are `docs/work-log.md` and the commit message.
- You do **not** call `Bash` except for: `git status`, `git log`, `git diff --stat`, `git diff --cached`, `git add <named-paths>`, `git commit`, the project's lint / test commands listed in **Common commands** of the spine (and only as a fallback if no verifier is available — which already means you are in degraded mode).
- You do **not** read source files except to derive plan-task context. If you need source-file context to make a decision, dispatch a fresh `task-implementer` or `code-reviewer` with the question — do not "just take a quick look" yourself.
- One task at a time. Never bundle.
- Never invoke unrelated agents (web search, doc lookup) — that is a sign the spine or plan is missing context. Stop and ask the caller.

## Parallelism rules

- **Within a task:** verifier + reviewer run in parallel. Always.
- **Across tasks:** do **not** run two implementers in parallel by default. The implementing context window is large, and disjoint scope is hard to prove ahead of time. If the caller explicitly opts in via `--parallel-implementers`, only dispatch concurrent implementers whose `Scope` paths are disjoint and whose `Depends on` chains do not overlap, and pass each implementer an explicit "your file scope only" instruction.
- **Reviewer concurrency:** if your project's review tooling has a known concurrency bug (e.g. a CLI review service that hangs on parallel sessions), serialize reviewers across tasks even when implementers ran in parallel. Document the tool name in the spine's hard rules.

## When to ask vs proceed

- No plan file → ask.
- No starting task ID and no green-committed task in the log → start at the first plan task; mention it in the first status line.
- Plan resolves to ≤ 10 tasks → proceed without asking.
- Plan resolves to 11–50 tasks → mention the count, proceed.
- Plan resolves to > 50 tasks → mention the count, ask for confirmation before starting.

## What this skill does NOT do

- Does not plan. Use `feature-planner` first.
- Does not audit completed tasks against current repo state. Use `check-completeness`.
- Does not push, tag, or open PRs. Commits only.
- Does not invent missing acceptance criteria. If a task lacks criteria, the loop stops and asks.
