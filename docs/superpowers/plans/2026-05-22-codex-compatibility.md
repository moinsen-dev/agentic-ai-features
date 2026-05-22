# Codex Compatibility Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `agentic-ai-features` installable from Codex while preserving Claude Code behavior.

**Architecture:** Keep one shared `skills/` tree and add Codex plugin metadata beside the existing Claude metadata. Update docs and skill instructions with a small platform-adaptation layer instead of forking the workflow.

**Tech Stack:** Markdown skills, JSON plugin manifests, Codex plugin validation script, git.

---

### Task 1: Add Codex Plugin Metadata

**Files:**
- Create: `.codex-plugin/plugin.json`
- Create: `.agents/plugins/marketplace.json`

- [x] **Step 1: Verify the missing manifest fails validation**

Run: `python3 /Users/udi/.codex/skills/.system/plugin-creator/scripts/validate_plugin.py /Users/udi/work/moinsen/opensource/agentic-ai-features`

Expected: FAIL with `missing .codex-plugin/plugin.json`.

- [x] **Step 2: Add the Codex plugin manifest**

Create `.codex-plugin/plugin.json` with `name`, `version`, `description`, author metadata, repo URLs, `skills: "./skills/"`, keywords, and Codex `interface` metadata.

- [x] **Step 3: Add the Codex marketplace descriptor**

Create `.agents/plugins/marketplace.json` with one local plugin entry pointing at `./`, `policy.installation: "AVAILABLE"`, `policy.authentication: "ON_INSTALL"`, and category `Coding`.

- [x] **Step 4: Validate the plugin manifest**

Run: `python3 /Users/udi/.codex/skills/.system/plugin-creator/scripts/validate_plugin.py /Users/udi/work/moinsen/opensource/agentic-ai-features`

Expected: PASS.

### Task 2: Add Codex Project Spine

**Files:**
- Create: `templates/AGENTS.md`
- Modify: `skills/init/SKILL.md`

- [x] **Step 1: Add `templates/AGENTS.md`**

Create a Codex-oriented spine with the same foundation gate, AI-feature rules, human gates, task format, refs map, and skill list as `templates/CLAUDE.md`, using Codex terminology and `AGENTS.md` scope semantics.

- [x] **Step 2: Update init instructions**

Update `skills/init/SKILL.md` so it says Claude Code writes `CLAUDE.md`, Codex writes `AGENTS.md`, and dual-environment projects may keep both files. Preserve the no-overwrite rule.

- [x] **Step 3: Inspect init wording**

Run: `rg -n "CLAUDE.md|AGENTS.md|Codex|Claude Code" skills/init/SKILL.md templates/AGENTS.md`

Expected: both project spine paths are described and no instruction tells Codex to write only `CLAUDE.md`.

### Task 3: Platform-Neutralize Workflow Docs

**Files:**
- Modify: `README.md`
- Modify: `skills/foundation/SKILL.md`
- Modify: `skills/feature-planner/SKILL.md`
- Modify: `skills/implement-task/SKILL.md`
- Modify: `skills/task-loop/SKILL.md`
- Modify: `skills/check-completeness/SKILL.md`

- [x] **Step 1: Update README install and use sections**

Add Codex install guidance, describe `.codex-plugin/plugin.json`, and clarify that Codex uses role briefs from `agents/*.md` when custom Claude Code sub-agent types are not available.

- [x] **Step 2: Add platform adaptation guidance to skills**

Add concise Claude Code and Codex equivalents for slash commands, project spine files, agent dispatch, and role-brief fallback. Keep the isolated implementer, verifier, and reviewer rule intact.

- [x] **Step 3: Check remaining platform-specific references**

Run: `rg -n "slash command|Agent tool|subagent_type|Claude Code|Codex" README.md skills templates`

Expected: remaining references either describe platform-specific behavior explicitly or are part of compatibility documentation.

### Task 4: Final Validation

**Files:**
- No new production files.

- [x] **Step 1: Run Codex plugin validation**

Run: `python3 /Users/udi/.codex/skills/.system/plugin-creator/scripts/validate_plugin.py /Users/udi/work/moinsen/opensource/agentic-ai-features`

Expected: PASS.

- [x] **Step 2: Validate JSON syntax**

Run: `python3 -m json.tool .codex-plugin/plugin.json >/dev/null && python3 -m json.tool .agents/plugins/marketplace.json >/dev/null`

Expected: exit 0.

- [x] **Step 3: Review changed files**

Run: `git diff --stat && git status --short`

Expected: only Codex compatibility files and docs are changed.
