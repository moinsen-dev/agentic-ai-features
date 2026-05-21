---
name: check-completeness
description: Audits whether planned or claimed work is supported by current repository evidence. Use for status sweeps, task completion checks, or before handoff. It does not write feature code.
---

# Check Completeness

You audit claims against evidence. You do not prove product correctness, and you do not treat another LLM's confidence as evidence.

## Inputs

The user must provide one of:

- a task ID;
- a range of task IDs;
- a plan file and scope;
- a "claimed done" list.

If no scope is provided, ask for one before auditing.

## Evidence Classes

Use three result classes:

| Result | Meaning |
|---|---|
| `PASS` | Concrete repo evidence or a command result satisfies the criterion. |
| `FAIL` | Evidence contradicts the criterion, is missing, or a required command fails. |
| `HUMAN` | The criterion requires subjective judgment, unavailable credentials, external services, hardware, visual approval, or product-owner confirmation. |

Do not mark `HUMAN` as `PASS`.

## Process

For each task or claim:

1. Read the task goal, scope, acceptance criteria, and verification commands.
2. Inspect the current repo state and relevant diffs or commits.
3. Run non-destructive verification commands when they are listed and available.
4. Produce one result per acceptance criterion.
5. Record missing or impossible verification as `FAIL` or `HUMAN`, not as success.

## Output

Return a concise audit report:

```markdown
## Completeness Audit

Scope: <task IDs or plan section>

| Item | Result | Evidence |
|---|---|---|
| <criterion> | PASS / FAIL / HUMAN | <command, file evidence, or reason> |

### Follow-ups
- <specific fix, verification, or human decision needed>
```

## Rules

- Never edit feature code.
- Do not broaden scope beyond the requested audit.
- Do not delete or rewrite task criteria during an audit.
- If you create audit files, keep them under `docs/audits/` and include the scope and timestamp.
- Prefer false negatives over false positives. A failed audit can be fixed; a false pass hides risk.

---

## Reachability Pass (orthogonal sub-routine)

The task-by-task audit above answers "does the diff for TASK-XXX exist?". The reachability pass answers a different question: **"is the new code actually reached from production entry points?"**

This catches the most common LLM-implementation bug: code added, tests written with injected fakes, build green, but the new class / method is never called from `main()` — the production path silently keeps the no-op. Tests cover their own fakes; production stays broken.

Run this pass:

- on user request — `reachability` as the scope argument;
- implicitly at the **end** of any plan-wide or phase-wide audit;
- before any milestone the user is treating as a release gate.

### Step R1 — Determine entry points

The user may provide entry points explicitly via `--entry-points <path,…>`. Otherwise, derive from the project's manifest:

- Node: `main` / `module` / `bin` fields in `package.json`; common defaults `src/index.ts`, `src/main.ts`, `dist/index.js`.
- Python: `[project.scripts]` or `setup.cfg [options.entry_points]` in `pyproject.toml`; common defaults `main.py`, `src/<pkg>/__main__.py`, `app.py`, `manage.py`.
- Rust: `[[bin]]` in `Cargo.toml`; common defaults `src/main.rs`, `src/bin/*.rs`.
- Dart / Flutter: `lib/main.dart`; entries in `pubspec.yaml` `executables:`.
- Go: `main` functions in `cmd/*/main.go` or top-level `main.go`.

If none of these resolve, **stop** and ask the user for the entry points. Do not guess.

Treat module-level side effects (`if __name__ == "__main__"`, top-level invocations in JS modules) as additional entry points.

### Step R2 — Enumerate candidates

A **candidate** is a symbol that the implementer intended to be wired into production but might have forgotten to wire. Three sources:

1. **Lifecycle methods** — methods whose names suggest production activation. Default regex (tune per project, document additions in the spine):
   ```
   ^\s*(?:async\s+)?(?:fn|def|void|Future|Task)\s*<?\w*>?\s+(initialize|start|boot|prime|enable[A-Z]\w*|register[A-Z]\w*|resume[A-Z]\w*|drain[A-Z]\w*|attach[A-Z]*|configure)\s*\(
   ```
2. **Production-flagged constructor parameters** — kwargs / parameters with a docstring or comment containing any of `production MUST inject`, `production wiring`, `@Production`, `# entry-point`, `// inject in main`. Walk back to the enclosing class.
3. **Top-level coordinator-style classes** — `class \w+(Coordinator|Drainer|Reconciler|Scheduler|Worker|Daemon|Listener)\b`. These often activate via construction (their constructor subscribes to streams or registers handlers).

The set of marker conventions is project-configurable. The spine's hard-rules section is the source of truth — if the project uses different markers, list them there and tune the grep here.

### Step R3 — Cross-reference

For each candidate `(symbol, defined-at)`, grep each entry-point file (and its transitive imports up to a reasonable depth — 2–3 levels) for the symbol's name. Status:

| Status | Meaning |
|---|---|
| `GROUNDED` | At least one production call site or constructor invocation found in an entry-point or in a file reachable from one. |
| `UNGROUNDED` | No production reference. The symbol compiles and its tests pass, but the runtime contract is broken. |
| `N/A` | Symbol is explicitly test-only (annotation, docstring, or path under a test directory). Record the evidence. |

### Step R4 — Emit findings

Write `docs/audits/reachability-<utc-iso>.md` with one row per candidate, grouped by status (UNGROUNDED first). Each UNGROUNDED row has a one-line "why this matters" — the runtime symptom the user will hit if the wiring stays missing.

If a previous reachability baseline exists, append a **delta** section listing newly UNGROUNDED / newly GROUNDED / removed candidates since last run. Deltas are the actionable signal on repeated runs.

### Step R5 — Do not file follow-up tasks automatically

The reachability pass reports; it does not file follow-up tasks. The user (or a deliberate `reachability file-followups` invocation) decides which UNGROUNDED rows become tasks.

---

## AI-Eval-Coverage Pass (sub-routine, runs alongside reachability when scope includes AI features)

Verifies the spine's "Eval before merge" hard rule: every prompt or LLM-output-shaped function has a corresponding eval artifact.

### Step E1 — Locate eval directory

Try, in order:

- `evals/`
- `test/evals/`
- `tests/evals/`
- `prompts/evals/`
- a path declared in the spine's hard-rules section

If none exist and the repo has any prompt assets, that is itself an `UNGROUNDED` finding — log and continue.

### Step E2 — Locate prompt / LLM-call assets

Grep the repo for:

- prompt string assignments — `(?i)(prompt|system_prompt|user_prompt)\s*=\s*["'`]`;
- file extensions for prompt assets — `*.prompt`, `*.prompt.md`, `*.tpl`, `*.jinja2`, files under `prompts/`;
- model-call functions — `anthropic.messages.create`, `openai.chat.completions.create`, `client.messages`, `model.generate`, `chat.completion`, `genkit`, `langchain.invoke`, etc. (tune per project; document additions in the spine).

### Step E3 — Match each asset to an eval

For each prompt asset or model-call site:

- find an eval file under the eval directory whose name references the same path or symbol — exact match, slug match, or explicit `<asset>: <eval-file>` mapping in `evals/index.md` if present;
- if found, status `GROUNDED`;
- if not, status `UNGROUNDED`.

### Step E4 — Emit findings

Append a `### AI-Eval-Coverage` section to the reachability audit file (same path). One row per asset. UNGROUNDED rows block any merge that touched that asset per the spine's hard rule.

---

## When to ask vs proceed

- Empty scope arg → **ask**.
- `all` over a plan with > 50 tasks → **warn + ask** (dispatch volume is high).
- Scope resolves to ≤ 10 tasks → proceed.
- `reachability` with no entry points derivable from manifests → **ask**.
- `reachability` and the repo has obvious entry points → proceed.

## Output to the user

While running, one terse line per audited task: `TASK-067 PASS` / `TASK-068 PARTIAL (3/5 PASS)` / `TASK-069 FAIL`. After the loop, the rollup table + the audit file path. For reachability passes, only emit the count per status + the file path — do not paste the full table.
