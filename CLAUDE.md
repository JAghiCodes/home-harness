# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This repo authors a **Claude Code plugin marketplace**. It is not an app — there is no build, no tests, no runtime. Edits under `core/` ship to end users via the `/plugin marketplace` system once published.

## Layout

- `.claude-plugin/marketplace.json` — suite-level manifest (lists `core` as a plugin)
- `core/` — the only published plugin
  - `.claude-plugin/plugin.json` — plugin manifest; the `skills` array must list every skill directory
  - `agents/*.md` · `commands/*.md` · `skills/*/SKILL.md` — component definitions
  - `hooks/hooks.json` — registers the PostToolUse hook for plan-file organisation
  - `scripts/rename-plan.sh`, `scripts/statusline.sh`
- `plans/` — dated plan folders; new plans land in `plans/.tmp/` and the hook moves them

See `README.md` for the user-facing harness overview and `core/README.md` for the full component catalogue.

## Authoring rules (load-bearing)

- **New skill** → create `core/skills/<name>/SKILL.md` **and** add `./skills/<name>` to the `skills` array in `core/.claude-plugin/plugin.json`. Skipping the second step silently disables the skill.
- **New agent / command** → drop a markdown file with `name` and `description` frontmatter into `core/agents/` or `core/commands/`. No registration step.
- **Hook edits** → `core/hooks/hooks.json` must follow the documented PostToolUse matcher shape. This was broken twice in recent history (commits `c9aa0b3`, `0fda92d`) — verify against current Claude Code hook docs before changing.
- **Plan files** → the PostToolUse hook captures them automatically. Two sources: (1) Claude Code plan-mode artifacts at `~/.claude/plans/<slug>.md` are mirrored into `plans/YYYY-MM-DD-<slug>/plan.md` here (this is the primary case — just use plan mode); (2) in-project drafts at `plans/.tmp/<slug>.md` get the same treatment. Both grow a `worklog.md` alongside. Don't create dated folders manually.
- **`${CLAUDE_PLUGIN_ROOT}` only expands in `hooks.json` and `.mcp.json`** — NOT in skill, agent or command markdown. For plugin-internal file refs in markdown use relative paths from the consuming file: `../` to plugin root from `agents/` or `commands/`, `../../` from `skills/<name>/`.
- **Subagents that produce files must declare `Write`** (or `Edit`, `NotebookEdit`) in their `tools:` frontmatter. `Read, Glob, Grep` alone does not grant file-write capability. Equally, a subagent that must run a command needs `Bash` — `BashOutput` without `Bash` lets it read the output of a shell it cannot start, which is how `code-reviewer` silently sat out every CI review until 2026-10-29.
- **Subagents are context-blind: they receive only their prompt, never the session conversation.** Any work that depends on conversation history — extracting lessons, judging *why* something changed, reading the user's corrections — must live in a main-thread skill or command, not a subagent. Fan out only for self-contained work, and hand each subagent a brief carrying the context it needs.
- **When porting prompts or schemas from another runtime**, strip every I/O-contract reference tied to that runtime: mount paths, registration steps, framework-specific orchestrator names, audit-directory conventions. "Verbatim" ports of contract-heavy files ship broken behaviour because the contract no longer holds.
- **Don't hardcode "today" or current-date strings** in agent prompts or reference content — they rot and drift between files. Instruct the agent to compute the gap from the current session date; Claude Code injects it into every session.
- **`github-project-tickets` anomaly** → this skill's `SKILL.md` lives one level deeper than all others: `core/skills/github-project-tickets/github-project-tickets/SKILL.md`. The plugin.json correctly points at `./skills/github-project-tickets`. Don't flatten it when editing — the nesting is intentional.

## Versioning

- Bump `core/.claude-plugin/plugin.json` `version` for plugin changes.
- **`marketplace.json` mirrors each plugin's version** in its `plugins[]` array — bumping `core/.claude-plugin/plugin.json` means bumping the matching `plugins[].version` in `.claude-plugin/marketplace.json`. These are the same fact in two files; let them drift and the marketplace advertises a stale version. This is separate from the suite-level `version` below, and it is now enforced by `core/scripts/lint-plugin-manifests.sh`.
- Bump `.claude-plugin/marketplace.json` `version` (the suite version) for suite-level changes (e.g. adding a second plugin).

## What this repo is not

- No `package.json`, no install step, no test runner. Don't suggest adding one.
- No CI runtime — just markdown + bash. The only lint signal is cSpell warnings (mostly false positives on British spelling and project terms).
