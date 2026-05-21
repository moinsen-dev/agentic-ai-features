---
name: init
description: Initialises a project with the agentic-ai-features spine. Copies the plugin's CLAUDE.md template into the current working directory, fills in basic placeholders from the repo (project name, author, repo URL), and stops without overwriting if a CLAUDE.md already exists. Use when starting a new project, or when retrofitting an existing project with the plugin's conventions.
---

# Init

You drop the `agentic-ai-features` project spine into the current working directory. One file: `CLAUDE.md` at the repo root.

## When to use

- The user just ran `/plugin install agentic-ai-features` in a new project and wants the spine wired up.
- An existing project has no CLAUDE.md and the user wants to adopt the plugin's conventions.
- The user explicitly says "init the plugin" or "drop the CLAUDE.md spine".

Do **not** use this skill to overwrite an existing CLAUDE.md. If one exists, stop and report — let the user decide whether to merge manually.

## Step 1 — Preflight

1. Confirm the cwd is a git repository (or that the user has acknowledged this is a project root). If not, ask.
2. Check whether `CLAUDE.md` already exists at the repo root. If it does, **stop** and emit a single line: `CLAUDE.md already exists at <path> — refusing to overwrite. Inspect it and decide whether to merge the template manually (template lives at <plugin>/templates/CLAUDE.md).`
3. Detect basic project facts from the repo for placeholder substitution:
   - Project name → from `package.json:name`, `pyproject.toml [project] name`, `Cargo.toml [package] name`, `pubspec.yaml name`, or fall back to the cwd's basename.
   - Description → from the same manifest's `description` field, or empty string.
   - Author / git user → from `git config user.name` and `git config user.email`.
   - Repo URL → from `git remote get-url origin` (strip `.git` suffix), or empty.

Do not invent values. Empty placeholders that you could not resolve from the repo stay as `<PLACEHOLDER>` so the user can fill them in.

## Step 2 — Locate the template

The template ships with this plugin at `templates/CLAUDE.md` relative to the plugin root. Read it. Do not edit the plugin's copy — read it as input and write the substituted version to the cwd.

If the template cannot be found (broken plugin install), stop with a clear error pointing at the expected path.

## Step 3 — Substitute and write

Apply these substitutions to the template content:

| Placeholder | Replacement |
|---|---|
| `<PROJECT-NAME>` | detected project name |
| `<one-line description>` | detected description, or `<one-line description>` if unknown |
| `<2-3 sentence project summary: what it is, who it is for, and what makes this project different. Replace this paragraph.>` | unchanged — the user writes this themselves |

Leave all `<Rule 1>`, `<Milestone 1>`, `<cmd>`, `<touching path X or behavior Y>`, etc. **unchanged**. Those are intentional fill-in markers; substituting them would hide work the user must still do.

Write the result to `<cwd>/CLAUDE.md`.

## Step 4 — Report

Emit a short report:

```
Wrote CLAUDE.md (<n> lines).
Substituted: project name = "<name>", author = "<git-user>"
Still placeholder: project summary, hard rules, common commands, refs trigger map.

Next steps:
- Open CLAUDE.md and replace the remaining placeholders.
- Add your project's specific hard rules (the AI-feature rules are already filled in).
- Add a row to "Common Commands" for each top-level command (build, test, lint, run).
- Skills are now available as /agentic-ai-features:feature-planner, /agentic-ai-features:task-loop, etc.
```

## Boundaries

- **Never overwrite an existing CLAUDE.md.** This is a hard rule, not a suggestion.
- Do not create `docs/`, `docs/refs/`, or any other supporting files. The template references them, but their content is project-specific — the user creates them as the project grows. The reachability pass in `check-completeness` will flag missing refs later, which is the right point to act.
- Do not run `git add` or `git commit`. Leave the file in the working tree for the user to inspect and commit.
- Do not detect or set up project-specific conventions (linting, testing frameworks). The template's "Hard Rules" section has placeholders for that; the user fills them in.
- One file only: `CLAUDE.md` at the cwd root. Nothing else.

## Failure modes

- **CLAUDE.md exists** → stop, do not overwrite, message the user.
- **Template missing** → stop, point at expected path, ask the user to reinstall the plugin.
- **Manifest detection ambiguous** (e.g. both `package.json` and `pyproject.toml` present) → use the basename of the cwd as the project name and proceed; note the ambiguity in the report.
- **No git repo** → still proceed if the user confirms; skip author / repo URL substitution.
