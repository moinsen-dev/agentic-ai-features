---
name: implement-task
description: Implements exactly one scoped task from this repository's plan. Use when the user asks to implement a specific TASK or the next approved task. Stops after the task is verified and reviewed; does not continue through the plan by default.
---

# Implement Task

You implement one task, verify it, and stop. This skill intentionally avoids long autonomous runs because LLMs can drift when task boundaries and acceptance criteria span too much context.

## Agent Isolation (non-negotiable)

The implementer, verifier, and reviewer steps below **must** each run in a separate Agent invocation via the Agent tool — fresh context, fresh prompt. **Do not** implement, verify, or review "directly" in the orchestrating context. The point of the three-step pipeline is three independent context windows; a self-review by the same agent that wrote the code defeats the purpose and rubber-stamps risk.

If a step's required agent is unavailable in this environment, stop and report `unable to dispatch` — do not silently fall back to running the rules yourself.

## Platform adaptation

- **Claude Code:** dispatch the named `agentic-ai-features:*` sub-agent types.
- **Codex:** use `multi_agent_v1.spawn_agent` only when the user explicitly authorized delegation. Include the matching role brief from `agents/task-implementer.md`, `agents/task-verifier.md`, or `agents/code-reviewer.md` in the prompt.
- **If Codex sub-agents are unavailable or unauthorized:** stop at a human gate. Do not collapse implementation, verification, and review into this same context.

## Inputs

- One task section in the repo task format.
- The project spine: `CLAUDE.md` or `AGENTS.md` if present, otherwise the matching template.
- Any referenced docs whose trigger matches the task.

If no single task is identified, ask the user which task to implement.

## Step 1 — Preflight

Before editing:

1. **Foundation gate.** If `README.md` is missing, OR `docs/foundation/OPEN-DECISIONS.md` is missing, OR `docs/foundation/OPEN-DECISIONS.md` contains any unchecked items (lines matching `- [ ]`), **stop** and emit: *"Foundation incomplete — run foundation first (`/agentic-ai-features:foundation` in Claude Code, `foundation` skill in Codex), then resolve every item in `docs/foundation/OPEN-DECISIONS.md` by ticking the checkbox after writing the decision inline."* Do not dispatch any agent.
2. Read the task section.
3. Extract `Goal`, `Scope`, `Out of scope`, `Depends on`, `Human gate`, `Acceptance criteria`, and `Verification`.
4. Confirm dependencies from repo evidence where possible.
5. If the task has a human gate that is not already cleared, stop and ask.
6. If the scope is ambiguous, stop and ask.

State the assumptions and the exact verification you will run.

## Step 2 — Implement

**Dispatch** the `agentic-ai-features:task-implementer` agent (defined at `agents/task-implementer.md` in this plugin) via the Agent tool. In Codex, dispatch a fresh sub-agent with `agents/task-implementer.md` included as the role brief. Pass the verbatim task section + the explicit instruction "Implement only this task; do not bundle; stage your edits but do not commit". The implementer must, in its own context:

- Change only files or behavior allowed by `Scope`.
- Not implement out-of-scope improvements.
- Stop and report the needed expansion if a required change falls outside scope.
- Keep edits minimal and aligned with existing repo style.
- Update project docs only when the task or self-improving rule requires it.

Do not implement in the orchestrating context.

## Step 3 — Verify

**Dispatch** the `agentic-ai-features:task-verifier` agent (defined at `agents/task-verifier.md` in this plugin) via the Agent tool. In Codex, dispatch a fresh sub-agent with `agents/task-verifier.md` included as the role brief. Pass the task's `Acceptance criteria` + `Verification` blocks. The verifier must, in its own context:

- Check every acceptance criterion against concrete evidence.
- Run the listed verification commands when available.
- Mark each item as `PASS`, `FAIL`, or `HUMAN`.
- Treat `HUMAN` as "requires subjective, unavailable, credentialed, hardware, visual, or product-owner judgment".

Do not convert `HUMAN` into `PASS`. Do not verify in the orchestrating context.

## Step 4 — Review

**Dispatch** the `agentic-ai-features:code-reviewer` agent (defined at `agents/code-reviewer.md` in this plugin) via the Agent tool. In Codex, dispatch a fresh sub-agent with `agents/code-reviewer.md` included as the role brief. Pass the task section + the implementer's summary. The reviewer must, in its own context, run the full review order from the code-reviewer role brief, including:

- scope creep
- failing or missing tests
- public interface or docs drift
- risky assumptions
- project convention violations
- unverifiable claims
- AI-feature checks (model pinning, prompt diff, eval coverage, cost budget) when the diff touches prompts or model calls

The reviewer can reduce risk. It cannot certify total product correctness. Do not review in the orchestrating context.

Steps 3 and 4 may be dispatched in parallel — both are read-only against the staged tree.

## Step 5 — Stop

After one task, return a short report containing:

- summary of changed files and behavior;
- verification results (one line per criterion, with result class);
- review outcome (`approved` / `approved with notes` / `blocked` / `unable to review`);
- list of any human follow-up;
- an explicit confirmation line: **"Implementer, verifier, and reviewer each ran in their own Agent invocation."**

Do not start the next task unless the user explicitly asks. For multi-task plans, use `task-loop` instead (`/agentic-ai-features:task-loop` in Claude Code).

## Completion Rule

The task is complete only when all non-human acceptance criteria pass and the review has no blocking items. If any item is `FAIL`, fix it or stop with a concrete blocker. If any item is `HUMAN`, stop and ask for the required human verification before claiming full completion.
