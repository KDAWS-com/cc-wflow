---
name: wflow:setup
description: >
  Set up an existing project with GitHub Issue templates, labels,
  GitLeaks pre-commit hook, and CLAUDE.md workflow section.
allowed-tools: Bash(gh issue *), Bash(gh pr *), Bash(gh label *), Bash(gh repo view *), Bash(gh repo edit *), Bash(gh api user), Bash(gh api user/orgs), Bash(gh auth status), Bash(gh --version), Bash(git add *), Bash(git commit *), Bash(git push *), Bash(git status), Bash(mkdir *), Bash(cp *), Bash(ls *), Bash(test *), Read, Write, Edit, Task
---

# wflow:setup — Workflow Setup

Set up an existing project for the GitHub Issue-driven workflow.

> **KDAWS** (kdaws.com) — part of the `wflow:` command set.

## Prerequisites

Before running, verify in order. Fail fast — stop on first failure with actionable guidance.

> **Note:** When invoked from `wflow:project-setup`, skip all prerequisites below (they were already verified).

1. **compound-engineering plugin installed** — Check `ls ~/.claude/plugins/cache/*/compound-engineering/skills/ >/dev/null 2>&1`. Required for stage commands.
2. **`gh` CLI installed** — Run `gh --version`.
3. **`gh` authenticated** — Run `gh auth status`.
4. **Connected to a GitHub repo** — Run `gh repo view --json nameWithOwner`. If no repo detected, offer to run `wflow:project-setup`. If user accepts, invoke it and return (project-setup will chain back, skipping these prerequisites).

## What it does

1. **Create issue templates** — Copies `feature.md`, `bug.md`, `refactor.md`, and `config.yml` to `.github/ISSUE_TEMPLATE/`
2. **Create labels** — Runs `gh label create` for `p1`, `p2`, `p3`, and `refactor`
3. **Enable auto-delete head branches** — Runs `gh repo edit` to enable branch cleanup after merge
4. **Install GitLeaks pre-commit hook** — Configures gitleaks to scan commits for secrets
5. **Add Workflow section to CLAUDE.md** — Appends a ~60-line Workflow section with pipeline overview, conventions, and command reference

## Steps

1. Check if `.github/ISSUE_TEMPLATE/` already exists. If so, ask: "Issue templates already exist. Overwrite?"
2. Copy templates from the skill's `templates/` directory:
   ```bash
   # Resolve the skill's installation directory
   SKILL_DIR=$(ls -d ~/.claude/plugins/cache/*/wflow/skills/github-workflow 2>/dev/null | head -1)
   # Fallback for local development
   if [ -z "$SKILL_DIR" ]; then SKILL_DIR="./skills/github-workflow"; fi

   mkdir -p .github/ISSUE_TEMPLATE
   cp "$SKILL_DIR/templates/feature.md" .github/ISSUE_TEMPLATE/
   cp "$SKILL_DIR/templates/bug.md" .github/ISSUE_TEMPLATE/
   cp "$SKILL_DIR/templates/refactor.md" .github/ISSUE_TEMPLATE/
   cp "$SKILL_DIR/templates/config.yml" .github/ISSUE_TEMPLATE/
   ```
3. Create labels (skip if they already exist):
   ```bash
   gh label create "p1" --color "d73a4a" --description "Critical priority" 2>/dev/null || true
   gh label create "p2" --color "f9a825" --description "Important priority" 2>/dev/null || true
   gh label create "p3" --color "0075ca" --description "Nice-to-have priority" 2>/dev/null || true
   gh label create "refactor" --color "cfd3d7" --description "Code restructuring" 2>/dev/null || true
   ```
4. Enable auto-delete head branches:
   ```bash
   gh repo edit --delete-branch-on-merge
   ```
5. Install GitLeaks pre-commit hook:
   1. Check if `gitleaks` is installed (`gitleaks version`)
   2. If not, ask permission to install (platform-appropriate: `brew install gitleaks`, `apt install gitleaks`, or download binary)
   3. Check if `.pre-commit-config.yaml` exists
      - If not: create with gitleaks hook config:
        ```yaml
        repos:
          - repo: https://github.com/gitleaks/gitleaks
            rev: 6eaad039603a4de39fddd1cf5f727391efe9974e  # v8.30.0
            hooks:
              - id: gitleaks
        ```
      - If exists: check if gitleaks already configured, add if missing
   4. Install pre-commit hooks (`pre-commit install` — install `pre-commit` itself with consent if needed)
   5. Report: "GitLeaks pre-commit hook installed. Future commits will be scanned for secrets."
6. Check if CLAUDE.md has a `## Workflow` section. If not, append the workflow section. If it already exists, ask: "Workflow section already exists in CLAUDE.md. Skip or overwrite?"

## CLAUDE.md Workflow Section

Append this to the project's CLAUDE.md (before `## Architecture` or at the end if no Architecture section exists):

```markdown
## Workflow

All development work is tracked via GitHub Issues. Every feature, bug, and refactor follows a defined pipeline.

### Pipeline

| Path | Stages | When |
|------|--------|------|
| **Full** | brainstorm → plan → (deepen-plan?) → technical review → work → review → security-review → compound | Multiple files, architectural decisions, new features |
| **Simple** | plan → work → security-review | Single-file changes, quick fixes (~30 min) |
| **Hotfix** | work → review → merge | P1 emergencies (run `quality:security-review` post-merge as follow-up) |

**Orchestrator commands:**
- `wflow:full #N` — Runs full pipeline, pauses before implementation for confirmation
- `wflow:simple #N` — Runs plan then implementation
- `quality:security-review` — Runs security scan with AI triage (standalone or within pipeline)

**Individual stage commands:**
- `/workflows:brainstorm #N`, `/workflows:plan #N`, `/deepen-plan`, `/technical_review #N`
- `/workflows:work #N`, `/workflows:review`, `/workflows:compound #N`
- `/resolve_pr_parallel`, `/triage`

Issue number (`#N`) is a **required argument** for all commands except `/workflows:review`, `quality:security-review`, `wflow:setup`, and `wflow:project-setup`.

### Issue Conventions

- **Title prefixes:** `feat:`, `fix:`, `refactor:` (conventional commits style)
- **Labels:** Type (`feature request`, `bug`, `refactor`) + Priority (`p1`, `p2`, `p3`)
- **Templates:** `.github/ISSUE_TEMPLATE/` — feature.md, bug.md, refactor.md
- **Note:** `feat:` prefix is intentionally short (follows Conventional Commits); the label is `feature request` (long form)

### Branch & PR Conventions

- **Branch naming:** `{type}/{issue-number}-{slug}` (e.g., `feat/42-osm-event-sync`)
- **Standardise on `feat/`** (not `feature/`)
- **PR title:** Matches issue title (e.g., `feat: OSM event sync`)
- **PR body:** Summary + `Closes #N` (auto-closes issue on merge)
- **Commits:** Conventional style with issue ref: `feat: add OAuth connection (#42)`
- **Draft PRs:** `/workflows:work` creates draft PRs. `/workflows:review` converts to ready-for-review.

### Stage Outputs

Each workflow stage posts a collapsible comment on the issue thread:

\```markdown
<!-- workflow-stage: plan, date: 2026-02-15, command: /workflows:plan -->
<details>
<summary>📋 Implementation Plan — 2026-02-15</summary>
[content]
</details>
\```

Emoji prefixes: 🔍 Brainstorm, 📋 Plan, 🔎 Technical Review, 🛡️ Security Review, 📚 Lessons Learned.

### Directory Status

| Directory | Status |
|-----------|--------|
| `todos/` | **Frozen** — no new files. Existing pending todos stay as-is. |
| `docs/brainstorms/` | For standalone exploration only (not tied to an issue) |
| `docs/plans/` | **Deprecated** — plans are now posted as issue comments, not stored locally |
| `docs/solutions/` | Active — `/workflows:compound` writes here |
```

## After Setup

Report what was created and suggest `wflow:full #N` and `wflow:simple #N` as next steps.
