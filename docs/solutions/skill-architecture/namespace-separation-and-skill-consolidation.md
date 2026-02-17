# Namespace Separation and Skill File Consolidation

**Date:** 2026-02-16
**Issue:** #1
**PR:** #2

## Problem

A Claude Code skill had orchestrator commands sharing a namespace (`/workflows:*`) with the external plugin commands they invoked. A separate reference file (`references/command-contracts.md`) duplicated ~60% of the main skill definition. Setup lacked prerequisite validation.

## Key Decisions

### 1. Two-tier namespace ownership

Orchestrator commands owned by the skill use `wflow:*`. Stage commands from the compound-engineering plugin keep their original names (`/workflows:*`, `/technical_review`, etc.). This makes ownership unambiguous for both humans and AI agents.

**Lesson:** When a skill orchestrates external plugin commands, use a distinct namespace prefix. Never share a namespace with your dependencies.

### 2. Single-file skill definitions

Merged `command-contracts.md` into `SKILL.md` and converted verbose per-command specs into a compact table. Result: 531 → 456 lines, zero duplication, single source of truth.

**Lesson:** For AI-consumed skill files, consolidation into one file is almost always better than splitting. The AI loads the entire skill in one read — cross-file references add latency and risk the agent missing context. Use compact tables over verbose per-item sections when all items share the same structure.

### 3. Prerequisite checks need explicit recursion guards

`wflow:project-setup` chains into `wflow:setup`, which can offer to run `wflow:project-setup`. Without an explicit guard, an AI agent could loop. The fix: a visible blockquote at the top of `wflow:setup` prerequisites saying "When invoked from `wflow:project-setup`, skip all prerequisites below."

**Lesson:** When two commands can invoke each other, add the recursion guard at the *entry point* of the callee, not just in the caller's instructions. AI agents need the guard visible at the point where they make the decision.

### 4. Validate user input in AI instruction docs

Shell command templates that interpolate user input (repo names, org names, issue numbers) need explicit validation rules in the skill definition. The AI agent may not independently sanitize inputs.

**Lesson:** Treat `{placeholder}` values in skill bash templates like untrusted input. Specify validation constraints (character sets, length limits) inline, even though this is instruction markdown, not executable code.

### 5. Progress output templates are unnecessary

AI agents naturally report status. Prescribed checkmark-emoji templates added 24 lines of cosmetic bloat. Removing them had zero functional impact.

**Lesson:** Don't over-specify presentation in skill files. Specify *what* to report, not *how* to format it. AI agents handle formatting well on their own.

## Review Process Insights

- Round 1 review (4 agents) found 2 fixable items — both addressed in minutes
- Round 2 review (4 agents) found 12 items — 11 accepted, 1 declined (broad `allowed-tools` is necessary for skill functionality)
- The simplicity reviewer's recommendation to convert verbose specs to a compact table was the highest-impact change (-40 lines)
- The security reviewer correctly identified that manual org name input lacked validation — easy to miss in instruction docs vs application code

## Gotchas

- `plan_review` was renamed to `technical_review` in compound-engineering 2.31.1. Old plugin versions may linger in the cache at `~/.claude/plugins/cache/`. Always check `installed_plugins.json` for the active version.
- Compound-engineering stage commands use inconsistent naming (underscores in `/technical_review`, kebab in `/deepen-plan`). This is upstream — document it, don't fight it.
- Git doesn't track empty directories. Deleting a file from a directory doesn't delete the directory — verify with `ls` or `rmdir`.
