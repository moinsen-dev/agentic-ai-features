# Codex Compatibility Design

Date: 2026-05-22

## Goal

Make `agentic-ai-features` installable and usable from both Claude Code and Codex while preserving the existing Claude Code workflow.

The project is currently shaped as a Claude Code plugin: `.claude-plugin/`, `skills/`, `agents/`, and `templates/`. The target state adds Codex plugin metadata and platform-neutral workflow text so Codex can discover the same skills without requiring Claude Code slash commands or custom Claude sub-agent types.

## Assumptions

- Claude Code compatibility remains a hard requirement.
- Codex compatibility should be additive, not a fork of the workflows.
- Codex should expose the existing `skills/` directory through a `.codex-plugin/plugin.json` manifest.
- Codex users should be able to install the plugin from a Codex marketplace entry.
- Custom role files under `agents/` remain useful in Codex as role briefs, even though Codex does not expose Claude Code `subagent_type` names from this plugin.
- This change does not add MCP servers, app connectors, hooks, or runtime services.

## Recommended Approach

Use one shared skill tree and add platform-specific metadata plus small wording changes:

- Add `.codex-plugin/plugin.json` with Codex plugin metadata and `skills: "./skills/"`.
- Add a Codex marketplace file at `.agents/plugins/marketplace.json`.
- Update documentation to describe both install paths: Claude Code marketplace/plugin install and Codex marketplace/plugin install.
- Update skill instructions so Claude Code remains first-class, while Codex gets explicit equivalents for slash commands, agent dispatch, and role-brief fallback.
- Add a Codex-oriented project spine template only if the existing `templates/CLAUDE.md` cannot cleanly express Codex usage without confusing Claude Code users.

This avoids a separate `codex-skills/` copy and keeps future workflow changes in one place.

## Architecture

### Plugin Metadata

`.claude-plugin/plugin.json` remains the Claude Code manifest. `.codex-plugin/plugin.json` becomes the Codex manifest.

The Codex manifest should include:

- `name`: `agentic-ai-features`
- `version`: current plugin semver
- `description`, `author`, `homepage`, `repository`, `license`, and `keywords`
- `skills`: `./skills/`
- `interface` metadata for Codex discovery

It should not include `mcpServers`, `apps`, or `hooks` because the repo does not provide those Codex companion files.

### Marketplace

Keep `.claude-plugin/marketplace.json` as the Claude Code marketplace descriptor.

Add `.agents/plugins/marketplace.json` using Codex marketplace shape:

- top-level marketplace `name`
- top-level `interface.displayName`
- `plugins[]` entry with `source.source: "local"` and `source.path: "./"`
- `policy.installation: "AVAILABLE"`
- `policy.authentication: "ON_INSTALL"`
- `category: "Coding"`

The source path should point at the plugin repo root because the manifest lives in the repo root under `.codex-plugin/`.

### Skill Text

The existing skills are reusable but currently mention Claude Code slash commands and custom `Agent` sub-agent names as if they are always available. The updated text should introduce a compact "Platform adaptation" rule:

- Claude Code: use the named slash command and custom agent type when available.
- Codex: use the skill body as the command equivalent, use `multi_agent_v1.spawn_agent` only when the user explicitly authorizes sub-agent work, and pass the relevant `agents/*.md` role brief as prompt context.
- If Codex sub-agents are unavailable or not authorized, stop at the human gate rather than silently self-reviewing.

This keeps the plugin's core rule intact: implementation, verification, and review should not be collapsed into one self-certifying context.

### Templates

`templates/CLAUDE.md` remains for Claude Code projects.

For Codex projects, either:

- add `templates/AGENTS.md` with the same foundation gate, AI-feature rules, human gates, and task format; or
- update `templates/CLAUDE.md` language to say Codex users should place equivalent rules in `AGENTS.md`.

The simpler and clearer option is to add `templates/AGENTS.md`. The `init` skill can then say:

- Claude Code writes `CLAUDE.md`.
- Codex writes `AGENTS.md`.
- If both environments are used, include both files or keep one canonical file and cross-reference it.

## Data Flow

Codex plugin discovery reads `.codex-plugin/plugin.json`, finds `skills/`, and loads skill metadata from each `SKILL.md`.

When a user asks for a workflow:

1. Codex triggers the relevant skill from `skills/*/SKILL.md`.
2. The skill checks the same foundation gates as Claude Code.
3. For planning and audit workflows, Codex executes the skill directly with normal repository tools.
4. For implementation pipelines, Codex either dispatches authorized sub-agents with the relevant role briefs or stops at the gate if isolation cannot be preserved.
5. Verification evidence remains command output, repository inspection, and explicit `PASS` / `FAIL` / `HUMAN` results.

## Error Handling

- Missing Codex marketplace entry: plugin can still be used from a direct local path, but install UX is incomplete.
- Missing `.codex-plugin/plugin.json`: Codex cannot discover the plugin as installable.
- Missing Codex sub-agent support or authorization: implementation skills stop with a human gate instead of self-reviewing.
- Skill text references an unavailable Claude Code command in Codex: the platform adaptation note must provide the Codex equivalent.
- AI-feature task lacks eval, model pinning, or budget criteria: existing hard gate remains unchanged.

## Testing

Verification should include:

- Validate `.codex-plugin/plugin.json` with the Codex plugin validator from the `plugin-creator` skill.
- Inspect `.agents/plugins/marketplace.json` for required Codex fields.
- Confirm `rg "Claude Code|slash command|Agent tool|subagent_type" skills templates README.md` only leaves references with a Codex adaptation nearby.
- Confirm existing Claude Code manifests remain unchanged unless explicitly needed.
- Confirm `git status --short` contains only intended files.

## Out Of Scope

- Rewriting the plugin as a Codex-only plugin.
- Adding MCP servers or app connectors.
- Changing the foundation council roles.
- Implementing an automated bridge that runs Claude Code slash commands from Codex.
- Adding product-specific AI evals for downstream apps.

## Success Criteria

- Codex can discover the plugin through `.codex-plugin/plugin.json`.
- Codex has a marketplace entry that points to this repo root.
- Claude Code install metadata still exists and remains valid.
- Skills describe how to run the workflows in both Claude Code and Codex.
- No workflow tells Codex to silently replace isolated implementer/verifier/reviewer roles with a single self-reviewing pass.
