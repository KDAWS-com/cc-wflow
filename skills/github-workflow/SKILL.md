---
description: >
  GitHub Issue-driven development workflow by KDAWS (kdaws.com).
  Detects code-creation intent and suggests the appropriate workflow.
  Shared conventions for branch naming, issue creation, and stage output formatting.
  Trigger: "work on issue", "start feature", "fix bug", describing new functionality or bugs.
---

# GitHub Issue-Driven Development Workflow — Shared Context

This skill provides detection logic and shared conventions used by the `wflow:` commands. It is auto-loaded as background context, not a slash command.

> **KDAWS** (kdaws.com) — the `wflow:` namespace identifies commands owned by this plugin.

## Available Commands

| Command | What it does |
|---------|-------------|
| `wflow:project-setup` | Bootstrap a new project: create repo, README, then run workflow setup |
| `wflow:setup` | Set up an existing project with templates, labels, and CLAUDE.md section |
| `wflow:full #N` | Run full pipeline: brainstorm → plan → (deepen?) → technical review → work → review → security-review → compound |
| `wflow:simple #N` | Run simple pipeline: plan → work → security-review |
| `quality:security-review` | Run security scan with AI triage — requires quality plugin |
| Individual stages *(compound-engineering plugin)* | `/workflows:brainstorm #N`, `/workflows:plan #N`, `/deepen-plan`, `/technical_review #N`, `/workflows:work #N`, `/workflows:review`, `/workflows:compound #N` |

Issue number (`#N`) is **required** for all commands except `/workflows:review`, `quality:security-review`, `wflow:setup`, and `wflow:project-setup`. Validate: digits only, positive integer.

## Git Safety

- **Only push to the `origin` remote.** Never push to any other remote.
- **Never use `--force` or `--delete` flags** with `git push`.
- **Branch protection rules** are recommended as a server-side backstop.

## Detection and Prompting

When you detect that the user is describing work that implies building or changing something, proactively suggest the workflow.

### Detection Signals

**Trigger on:**
- Explicit requests: "add a feature", "fix this bug", "refactor the...", "build a...", "implement..."
- Implicit intent: describing desired behaviour changes, pointing out broken functionality, proposing improvements

**Do NOT trigger on:**
- Research questions: "how does X work?", "what does this code do?"
- File exploration: "show me the routes", "find the controller for..."
- Configuration queries: "what version of PHP are we using?"
- Already mid-workflow on the same issue (on a `{type}/{N}-slug` branch or branch has an open PR)

### Detection Prompt

When triggered, present this prompt:

```
This looks like a [feature/bug/refactor]:

  Title: "[type]: [description inferred from user's message]"
  Labels: [appropriate type label], p2
  Workflow: [full/simple] ([reasoning])

Options:
  1. Run full workflow (wflow:full — brainstorm → plan → technical review → implement → code review → security review → learn)
  2. Run simple workflow (wflow:simple — plan → implement → security review)
  3. Just create the issue for later
  4. Skip workflow, work directly
```

### Heuristics for Simple vs Full

**Suggest simple when:**
- Single-file change likely
- User says "quick fix", "small bug", "minor tweak"
- The fix is obvious from the description (no design decisions needed)

**Suggest full when:**
- Multiple files or new models/components
- Unclear requirements needing exploration
- Architectural decisions involved
- User describes a new feature

### After User Chooses

- **Option 1 (full):** Create the issue via `gh issue create`, then run `wflow:full #N`
- **Option 2 (simple):** Create the issue via `gh issue create`, then run `wflow:simple #N`
- **Option 3 (create only):** Create the issue, report the URL, stop
- **Option 4 (skip):** Proceed without workflow. Do not suggest again for the same topic.

**Priority:** Default to `p2`. If user specifies urgency, adjust accordingly.

**Title prefix:** If user's description lacks a type prefix, suggest one: "I'll create this as `feat: Add PDF export`. Correct?"

## Issue Creation

When creating issues (from detection prompt or manually):

```bash
# Feature
gh issue create --title "feat: Description" --label "feature request" --label "p2" \
  --body "## Problem\n\n[description]\n\n## Goal\n\n[goal]\n\n## Acceptance Criteria\n\n- [ ] ...\n\n**Priority:** P2"

# Bug
gh issue create --title "fix: Description" --label "bug" --label "p1" \
  --body "## Bug Description\n\n[description]\n\n## Steps to Reproduce\n\n1. ...\n\n**Priority:** P1"

# Refactor
gh issue create --title "refactor: Description" --label "refactor" --label "p3" \
  --body "## Current State\n\n[description]\n\n## Goal\n\n[goal]\n\n**Priority:** P3"
```

## Branch Naming

When `/workflows:work` creates a branch:

```
{type}/{issue-number}-{slug}
```

- `type`: `feat`, `fix`, or `refactor` (extracted from issue title prefix)
- `issue-number`: The issue number (e.g., `42`)
- `slug`: Kebab-case summary (e.g., `osm-event-sync`)

Examples: `feat/42-osm-event-sync`, `fix/87-broken-login`, `refactor/100-extract-service`

## Draft PRs

`/workflows:work` creates PRs as **drafts** (`gh pr create --draft`). This signals "work in progress, not ready for review".

- `/workflows:review` converts the draft to ready-for-review via `gh pr ready` before posting review comments
- Manual review is also possible — the draft status is a signal, not a gate

## Collapsible Comment Format

All stage outputs posted to issues use this format:

```markdown
<!-- workflow-stage: {stage}, date: {YYYY-MM-DD}, command: {command} -->
<details>
<summary>{emoji} {Stage Name} — {YYYY-MM-DD}</summary>

{stage output}

---
*Generated by `{command}` • Based on {input source}*
</details>
```

### Posting Comments

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
gh issue comment N --body "$(cat <<'EOF'
<!-- workflow-stage: plan, date: 2026-02-15, command: /workflows:plan -->
<details>
<summary>📋 Implementation Plan — 2026-02-15</summary>

[plan content here]

---
*Generated by `/workflows:plan` • Based on issue body + brainstorm*
</details>
EOF
)"
```

### Reading Prior Stage Outputs

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
# Find most recent plan comment on issue #42
gh api repos/${REPO}/issues/42/comments \
  --jq '[.[] | select(.body | contains("<!-- workflow-stage: plan"))] | last | .body'
```

Use `<!-- workflow-stage: {stage} -->` as the machine-readable marker. The emoji is for human readability; parsing uses the HTML comment.
