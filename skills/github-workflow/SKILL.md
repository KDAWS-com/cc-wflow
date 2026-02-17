---
description: >
  GitHub Issue-driven development workflow by KDAWS (kdaws.com).
  Orchestrates full and simple pipelines with cross-plugin security review. Detects code-creation intent and suggests the appropriate workflow.
  Commands: wflow:full, wflow:simple, wflow:setup, wflow:project-setup.
  Trigger: "work on issue", "start feature", "fix bug", describing new functionality or bugs.
argument-hint: "#issue-number"
allowed-tools: Bash(gh issue *), Bash(gh pr *), Bash(gh label *), Bash(gh repo create *), Bash(gh repo view *), Bash(gh repo edit *), Bash(gh api user), Bash(gh api user/orgs), Bash(gh api repos/*/issues/*/comments *), Bash(gh auth status), Bash(gh --version), Bash(git init), Bash(git add *), Bash(git commit *), Bash(git merge-base *), Bash(git push *), Bash(git checkout *), Bash(git branch *), Bash(git status), Bash(git diff *), Bash(git log *), Bash(mkdir *), Bash(cp *), Bash(ls *), Bash(test *), Read, Write, Edit, Task, Skill
---

# GitHub Issue-Driven Development Workflow

Every piece of work starts as a GitHub Issue and follows a defined pipeline. This skill provides orchestrator commands that chain stages automatically, and detection logic that suggests the workflow when you describe work.

> **KDAWS** (kdaws.com) — the `wflow:` namespace identifies commands owned by this skill.

## Quick Reference

| Command | What it does |
|---------|-------------|
| `wflow:project-setup` | Bootstrap a new project: create repo, README, then run workflow setup |
| `wflow:setup` | Set up an existing project with templates, labels, and CLAUDE.md section |
| `wflow:full #N` | Run full pipeline: brainstorm → plan → (deepen?) → technical review → work → review → security-review → compound |
| `wflow:simple #N` | Run simple pipeline: plan → work → security-review |
| `quality:security-review` | Run security scan with AI triage (standalone or within pipeline) — requires quality plugin |
| Individual stages *(compound-engineering plugin)* | `/workflows:brainstorm #N`, `/workflows:plan #N`, `/deepen-plan`, `/technical_review #N`, `/workflows:work #N`, `/workflows:review`, `/workflows:compound #N` |

Issue number (`#N`) is **required** for all commands except `/workflows:review`, `quality:security-review`, `wflow:setup`, and `wflow:project-setup`. Validate: digits only, positive integer.

## File Structure

This skill uses progressive disclosure to stay under the plugin skill size limit. Before executing a command, load the relevant reference file from this skill's directory using the Read tool:

| Command | Load first |
|---------|-----------|
| `wflow:full #N` or `wflow:simple #N` | `orchestrators.md` in this directory |
| Error handling or rework paths | `reference.md` in this directory |

To resolve the skill's directory path:
```bash
SKILL_DIR=$(ls -d ~/.claude/plugins/cache/*/wflow/skills/github-workflow 2>/dev/null | head -1)
```

If `SKILL_DIR` is empty (skill loaded from local development, not a plugin cache):
```bash
SKILL_DIR="./skills/github-workflow"
```

> **Note:** This split departs from the issue #1 single-file consolidation learning, justified by the 500-line plugin skill constraint that did not exist when that learning was documented.

> **Known limitation:** The `SKILL_DIR` glob resolution assumes a single cached version of the plugin. If multiple versions exist in the cache, `head -1` picks whichever sorts first alphabetically. This is acceptable for v1.0.0 — revisit when Claude Code provides native skill-relative path resolution.

## Git Safety

- **Only push to the `origin` remote.** Never push to any other remote.
- **Never use `--force` or `--delete` flags** with `git push`.
- **Branch protection rules** are recommended as a server-side backstop. Enable via: `Settings → Branches → Add rule → Require PR reviews before merging` for `main`.

## Project Setup: `wflow:project-setup`

Bootstrap a new project from scratch. Creates a local repo, remote GitHub repo, generates a README, then auto-chains into `wflow:setup`.

### Prerequisites

Before running, verify in order. Fail fast — stop on first failure with actionable guidance.

1. **compound-engineering plugin installed** — Check `ls ~/.claude/plugins/cache/*/compound-engineering/skills/ >/dev/null 2>&1`. Required for stage commands.
2. **`gh` CLI installed** — Run `gh --version`.
3. **`gh` authenticated** — Run `gh auth status`.

### Steps

1. **Ask for project description** — Use AskUserQuestion: "What does this project do? Describe it in 1-2 sentences."

2. **Generate repo name** — Generate a kebab-case repo name from the description (2-4 words). Present to user: "Suggested repo name: `{name}`. Use this name?" If user rejects, ask them to provide a preferred name. Validate: lowercase, hyphens only, no leading/trailing hyphens, 1-100 chars.

3. **Choose owner** — Query available targets:
   ```bash
   gh api user --jq '.login'
   gh api user/orgs --jq '.[].login'
   ```
   Present as numbered list:
   ```
   Select repository owner:
   1. {username} (personal)
   2. {org-1}
   3. {org-2}
   ```
   If org listing fails or returns empty, offer personal account with option to type an org name manually. Validate any manually-typed owner name: alphanumeric and hyphens only, cannot start with a hyphen, max 39 chars (GitHub username constraints).

4. **Check for collisions** — Before creating anything:
   ```bash
   # Local collision
   test -d ./{repo-name} && echo "EXISTS"
   # Remote collision
   gh repo view {owner}/{repo-name} --json name --jq '.name' 2>/dev/null
   ```
   **Local collision:** "Directory `./{name}` already exists. Choose a different name or remove the existing directory." Loop back to name selection.
   **Remote collision:** "Repository `{owner}/{name}` already exists on GitHub. Choose a different name." Loop back to name selection.

5. **Create local project:**
   ```bash
   mkdir {repo-name}
   cd {repo-name}
   git init
   ```

6. **Generate README.md:**
   ```markdown
   # {Repo Name (title case)}

   {User's description}

   ## Setup

   _Coming soon._

   ## License

   _TBD_
   ```

7. **Initial commit:**
   ```bash
   git add README.md
   git commit -m "feat: initial project setup"
   ```

8. **Create remote and push:**
   ```bash
   gh repo create {owner}/{repo-name} --private --source=. --push
   ```
   **If fails:** Leave local directory intact. Report error with manual recovery instructions including the commands to retry.

9. **Auto-chain into `wflow:setup`** — Announce: "Project created. Running workflow setup..." Then execute `wflow:setup`, skipping its prerequisite checks (they were already verified in step 1-3 and the repo was just created).

### Repository Context Resolution

All commands that use `gh api` should first resolve the repo:
```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
# Use ${REPO} in all gh api calls: gh api repos/${REPO}/issues/...
```

## Workflow Setup: `wflow:setup`

Set up an existing project for the GitHub Issue-driven workflow.

### Prerequisites

Before running, verify in order. Fail fast — stop on first failure with actionable guidance.

> **Note:** When invoked from `wflow:project-setup`, skip all prerequisites below (they were already verified).

1. **compound-engineering plugin installed** — Check `ls ~/.claude/plugins/cache/*/compound-engineering/skills/ >/dev/null 2>&1`. Required for stage commands.
2. **`gh` CLI installed** — Run `gh --version`.
3. **`gh` authenticated** — Run `gh auth status`.
4. **Connected to a GitHub repo** — Run `gh repo view --json nameWithOwner`. If no repo detected, offer to run `wflow:project-setup`. If user accepts, invoke it and return (project-setup will chain back, skipping these prerequisites).

### What it does

1. **Create issue templates** — Copies `feature.md`, `bug.md`, `refactor.md`, and `config.yml` to `.github/ISSUE_TEMPLATE/`
2. **Create labels** — Runs `gh label create` for `p1`, `p2`, `p3`, and `refactor`
3. **Enable auto-delete head branches** — Runs `gh repo edit` to enable branch cleanup after merge
4. **Install GitLeaks pre-commit hook** — Configures gitleaks to scan commits for secrets
5. **Add Workflow section to CLAUDE.md** — Appends a ~60-line Workflow section with pipeline overview, conventions, and command reference

### Steps

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

### CLAUDE.md Workflow Section

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

```markdown
<!-- workflow-stage: plan, date: 2026-02-15, command: /workflows:plan -->
<details>
<summary>📋 Implementation Plan — 2026-02-15</summary>
[content]
</details>
```

Emoji prefixes: 🔍 Brainstorm, 📋 Plan, 🔎 Technical Review, 🛡️ Security Review, 📚 Lessons Learned.

### Directory Status

| Directory | Status |
|-----------|--------|
| `todos/` | **Frozen** — no new files. Existing pending todos stay as-is. |
| `docs/brainstorms/` | For standalone exploration only (not tied to an issue) |
| `docs/plans/` | **Deprecated** — plans are now posted as issue comments, not stored locally |
| `docs/solutions/` | Active — `/workflows:compound` writes here |
```

### After Init

Report what was created and suggest `wflow:full #N` and `wflow:simple #N` as next steps.

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

Examples:
- `feat/42-osm-event-sync`
- `fix/87-broken-login`
- `refactor/100-extract-service`

## Draft PRs

`/workflows:work` creates PRs as **drafts** (`gh pr create --draft`). This signals "work in progress, not ready for review" and prevents accidental early reviews.

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
