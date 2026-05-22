---
name: init
description: Initialises a project with the agentic-ai-features spine. Copies the platform-appropriate template into the current working directory (`CLAUDE.md` for Claude Code, `AGENTS.md` for Codex), fills in basic placeholders from the repo, and stops without overwriting an existing spine file. Use when starting a new project, or when retrofitting an existing project with the plugin's conventions.
---

# Init

You drop the `agentic-ai-features` project spine into the current working directory. Use `CLAUDE.md` for Claude Code projects and `AGENTS.md` for Codex projects. If the user wants both platforms in the same project, write both files only when neither target file already exists.

## When to use

- The user just installed `agentic-ai-features` in a new project and wants the spine wired up.
- An existing project has no platform spine and the user wants to adopt the plugin's conventions.
- The user explicitly says "init the plugin", "drop the CLAUDE.md spine", or "drop the AGENTS.md spine".

Do **not** use this skill to overwrite an existing `CLAUDE.md` or `AGENTS.md`. If the target file exists, stop and report — let the user decide whether to merge manually.

## Step 1 — Preflight

1. Confirm the cwd is a git repository (or that the user has acknowledged this is a project root). If not, ask.
2. Determine the platform target:
   - Claude Code → target file `CLAUDE.md`, template `templates/CLAUDE.md`.
   - Codex → target file `AGENTS.md`, template `templates/AGENTS.md`.
   - If both are requested → process `CLAUDE.md` first, then `AGENTS.md`, applying the same no-overwrite rule to each file.
   - If the platform is unclear, ask which spine to write.
3. Check whether the target file already exists at the repo root. If it does, **stop** and emit a single line: `<target> already exists at <path> — refusing to overwrite. Inspect it and decide whether to merge the template manually (template lives at <plugin>/<template>).`
4. Detect basic project facts from the repo for placeholder substitution:
   - Project name → from `package.json:name`, `pyproject.toml [project] name`, `Cargo.toml [package] name`, `pubspec.yaml name`, or fall back to the cwd's basename.
   - Description → from the same manifest's `description` field, or empty string.
   - Author / git user → from `git config user.name` and `git config user.email`.
   - Repo URL → from `git remote get-url origin` (strip `.git` suffix), or empty.

Do not invent values. Empty placeholders that you could not resolve from the repo stay as `<PLACEHOLDER>` so the user can fill them in.

## Step 2 — Locate the template

The templates ship with this plugin at `templates/CLAUDE.md` and `templates/AGENTS.md` relative to the plugin root. Read the selected template. Do not edit the plugin's copy — read it as input and write the substituted version to the cwd.

If the template cannot be found (broken plugin install), stop with a clear error pointing at the expected path.

## Step 3 — Substitute and write

Apply these substitutions to the template content:

| Placeholder | Replacement |
|---|---|
| `<PROJECT-NAME>` | detected project name |
| `<one-line description>` | detected description, or `<one-line description>` if unknown |
| `<2-3 sentence project summary: what it is, who it is for, and what makes this project different. Replace this paragraph.>` | unchanged — the user writes this themselves |

Leave all `<Rule 1>`, `<Milestone 1>`, `<cmd>`, `<touching path X or behavior Y>`, etc. **unchanged**. Those are intentional fill-in markers; substituting them would hide work the user must still do.

Write the result to the selected target file at the cwd root.

## Step 4 — Report

Emit a short report:

```
Wrote <target> (<n> lines).
Substituted: project name = "<name>", author = "<git-user>"
Still placeholder until foundation runs: project summary, hard rules, common commands, refs trigger map.

Next steps:
- Optionally fill any obvious project summary now. The `foundation` workflow will hydrate the spine from council output after synthesis.
- Run the `foundation` workflow next. In Claude Code, use `/agentic-ai-features:foundation`; in Codex, invoke the `foundation` skill. It runs a 5-perspective council, writes README.md plus docs/foundation/, then hydrates the spine and docs/refs/. Until that has run and its OPEN-DECISIONS are resolved, `feature-planner`, `implement-task`, and `task-loop` refuse to start.
- Skills are now available through `/agentic-ai-features:*` in Claude Code and through the Codex plugin skill list in Codex.
```

## Boundaries

- **Never overwrite an existing `CLAUDE.md` or `AGENTS.md`.** This is a hard rule, not a suggestion.
- Do not create `docs/`, `docs/refs/`, or any other supporting files. The template references them, but their content is project-specific — the `foundation` workflow hydrates them after the council and synthesizer have produced project-specific context.
- Do not run `git add` or `git commit`. Leave the file in the working tree for the user to inspect and commit.
- Do not detect or set up project-specific conventions (linting, testing frameworks). The template's "Hard Rules" section has placeholders for that; the user fills them in.
- One target spine file only unless the user explicitly asks for both platform spines.

## Failure modes

- **Target spine file exists** → stop, do not overwrite, message the user.
- **Template missing** → stop, point at expected path, ask the user to reinstall the plugin.
- **Manifest detection ambiguous** (e.g. both `package.json` and `pyproject.toml` present) → use the basename of the cwd as the project name and proceed; note the ambiguity in the report.
- **No git repo** → still proceed if the user confirms; skip author / repo URL substitution.
