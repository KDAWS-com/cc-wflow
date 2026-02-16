---
description: >
  GitHub Issue-driven development workflow by KDAWS (kdaws.com).
  Orchestrates full and simple pipelines with integrated security review. Detects code-creation intent and suggests the appropriate workflow.
  Commands: kdaws:wf-full, kdaws:wf-simple, kdaws:wf-setup, kdaws:project-setup, kdaws:security-review.
  Trigger: "work on issue", "start feature", "fix bug", describing new functionality or bugs.
argument-hint: "#issue-number"
allowed-tools: Bash(gh *), Bash(git *), Bash(semgrep scan *), Bash(phpstan analyse *), Bash(composer audit *), Bash(govulncheck *), Bash(gosec *), Bash(gitleaks detect *), Bash(gitleaks protect *), Read, Write, Edit, Task, Skill
---

# GitHub Issue-Driven Development Workflow

Every piece of work starts as a GitHub Issue and follows a defined pipeline. This skill provides orchestrator commands that chain stages automatically, and detection logic that suggests the workflow when you describe work.

> **KDAWS** (kdaws.com) — the `kdaws:` namespace identifies commands owned by this skill.

## Setup

Run `kdaws:wf-setup` to set up an existing project. Run `kdaws:project-setup` to bootstrap a new project from scratch. See sections below.

## Quick Reference

| Command | What it does |
|---------|-------------|
| `kdaws:project-setup` | Bootstrap a new project: create repo, README, then run workflow setup |
| `kdaws:wf-setup` | Set up an existing project with templates, labels, and CLAUDE.md section |
| `kdaws:wf-full #N` | Run full pipeline: brainstorm → plan → (deepen?) → technical review → work → review → security-review → compound |
| `kdaws:wf-simple #N` | Run simple pipeline: plan → work → security-review |
| `kdaws:security-review` | Run security scan with AI triage (standalone or within pipeline) |
| Individual stages *(compound-engineering plugin)* | `/workflows:brainstorm #N`, `/workflows:plan #N`, `/deepen-plan`, `/technical_review #N`, `/workflows:work #N`, `/workflows:review`, `/workflows:compound #N` |

Issue number (`#N`) is **required** for all commands except `/workflows:review`, `kdaws:security-review`, `kdaws:wf-setup`, and `kdaws:project-setup`. Validate: digits only, positive integer.

## Project Setup: `kdaws:project-setup`

Bootstrap a new project from scratch. Creates a local repo, remote GitHub repo, generates a README, then auto-chains into `kdaws:wf-setup`.

### Prerequisites

Before running, verify in order. Fail fast — stop on first failure with actionable guidance.

1. **compound-engineering plugin installed** — Check `~/.claude/plugins/cache/every-marketplace/compound-engineering/` exists. Required for stage commands.
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
   gh api repos/{owner}/{repo-name} --jq '.name' 2>/dev/null
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

9. **Auto-chain into `kdaws:wf-setup`** — Announce: "Project created. Running workflow setup..." Then execute `kdaws:wf-setup`, skipping its prerequisite checks (they were already verified in step 1-3 and the repo was just created).

## Workflow Setup: `kdaws:wf-setup`

Set up an existing project for the GitHub Issue-driven workflow.

### Prerequisites

Before running, verify in order. Fail fast — stop on first failure with actionable guidance.

> **Note:** When invoked from `kdaws:project-setup`, skip all prerequisites below (they were already verified).

1. **compound-engineering plugin installed** — Check `~/.claude/plugins/cache/every-marketplace/compound-engineering/` exists. Required for stage commands.
2. **`gh` CLI installed** — Run `gh --version`.
3. **`gh` authenticated** — Run `gh auth status`.
4. **Connected to a GitHub repo** — Run `gh repo view --json nameWithOwner`. If no repo detected, offer to run `kdaws:project-setup`. If user accepts, invoke it and return (project-setup will chain back, skipping these prerequisites).

### What it does

1. **Create issue templates** — Copies `feature.md`, `bug.md`, `refactor.md`, and `config.yml` to `.github/ISSUE_TEMPLATE/`
2. **Create labels** — Runs `gh label create` for `p1`, `p2`, `p3`, and `refactor`
3. **Enable auto-delete head branches** — Runs `gh api` to enable branch cleanup after merge
4. **Install GitLeaks pre-commit hook** — Configures gitleaks to scan commits for secrets
5. **Add Workflow section to CLAUDE.md** — Appends a ~60-line Workflow section with pipeline overview, conventions, and command reference

### Steps

1. Check if `.github/ISSUE_TEMPLATE/` already exists. If so, ask: "Issue templates already exist. Overwrite?"
2. Copy templates from the skill's `templates/` directory:
   ```bash
   # Use the skill's installed directory (the directory containing this SKILL.md file)
   mkdir -p .github/ISSUE_TEMPLATE
   cp "${SKILL_DIR}/templates/feature.md" .github/ISSUE_TEMPLATE/
   cp "${SKILL_DIR}/templates/bug.md" .github/ISSUE_TEMPLATE/
   cp "${SKILL_DIR}/templates/refactor.md" .github/ISSUE_TEMPLATE/
   cp "${SKILL_DIR}/templates/config.yml" .github/ISSUE_TEMPLATE/
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
   gh api repos/{owner}/{repo} -X PATCH -f delete_branch_on_merge=true
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
| **Hotfix** | work → review → merge | P1 emergencies (run `kdaws:security-review` post-merge as follow-up) |

**Orchestrator commands:**
- `kdaws:wf-full #N` — Runs full pipeline, pauses before implementation for confirmation
- `kdaws:wf-simple #N` — Runs plan then implementation
- `kdaws:security-review` — Runs security scan with AI triage (standalone or within pipeline)

**Individual stage commands:**
- `/workflows:brainstorm #N`, `/workflows:plan #N`, `/deepen-plan`, `/technical_review #N`
- `/workflows:work #N`, `/workflows:review`, `/workflows:compound #N`
- `/resolve_pr_parallel`, `/triage`

Issue number (`#N`) is a **required argument** for all commands except `/workflows:review` and `kdaws:security-review`.

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

Report what was created and suggest `kdaws:wf-full #N` and `kdaws:wf-simple #N` as next steps.

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
  1. Run full workflow (kdaws:wf-full — brainstorm → plan → technical review → implement → code review → security review → learn)
  2. Run simple workflow (kdaws:wf-simple — plan → implement → security review)
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

- **Option 1 (full):** Create the issue via `gh issue create`, then run `kdaws:wf-full #N`
- **Option 2 (simple):** Create the issue via `gh issue create`, then run `kdaws:wf-simple #N`
- **Option 3 (create only):** Create the issue, report the URL, stop
- **Option 4 (skip):** Proceed without workflow. Do not suggest again for the same topic.

**Priority:** Default to `p2`. If user specifies urgency, adjust accordingly.

**Title prefix:** If user's description lacks a type prefix, suggest one: "I'll create this as `feat: Add PDF export`. Correct?"

## Orchestrator: `kdaws:wf-full #N`

Run the complete pipeline for issue `#N`.

### Steps

1. **Read issue** — `gh issue view N --json title,body,labels`
2. **Brainstorm** — Invoke `/workflows:brainstorm #N`. Post output as collapsible comment on issue.
3. **Plan** — Invoke `/workflows:plan #N`. Post output as collapsible comment on issue.
4. **Deepen plan (optional)** — Ask: "Deepen this plan with parallel research? (recommended for complex features)". If yes, invoke `/deepen-plan` on the plan file, then post the deepened plan as a new collapsible comment on the issue (stage: `plan`, replacing the previous plan as the "most recent").
5. **Technical review** — Invoke `/technical_review #N`. Post output as collapsible comment on issue.
6. **Confirm before implementation** — Ask: "Plan reviewed. Ready to start coding? (review the plan on the issue thread first)". Wait for user confirmation.
7. **Work** — Invoke `/workflows:work #N`. Creates branch and **draft** PR.
8. **Review** — Convert to ready-for-review via `gh pr ready`, then invoke `/workflows:review`. Posts review comments on PR.
9. **Resolve fixes** — If review has findings, see [Rework and Rejection Paths](#rework-and-rejection-paths).
10. **Security Review** — Invoke `kdaws:security-review`. If blocking findings:
    - Fix findings directly on the current branch, then re-run `kdaws:security-review`.
    - Max 3 security review cycles. After 3rd failure, escalate to manual review.
    - If clean → proceed to compound.
11. **Compound** — After merge, invoke `/workflows:compound #N`. Documents lessons learned.

### Between Stages

- Post each stage's output to the GitHub issue as a collapsible comment (see format below). Exception: `kdaws:security-review` posts to the PR.
- The issue thread is the shared context — each stage reads prior outputs from GitHub
- User can stop the pipeline at any point by saying "stop" or "pause"
- If any stage fails, stop and report the error

## Orchestrator: `kdaws:wf-simple #N`

Run the quick pipeline for issue `#N`.

### Steps

1. **Read issue** — `gh issue view N --json title,body,labels`
2. **Plan** — Invoke `/workflows:plan #N`. Post output as collapsible comment on issue.
3. **Confirm before implementation** — Ask: "Plan ready. Start coding?". Wait for user confirmation.
4. **Work** — Invoke `/workflows:work #N`. Creates branch and **draft** PR.
5. **Security Review** — Invoke `kdaws:security-review`. Fix findings on current branch and re-run. Max 3 cycles, then escalate.

No brainstorm, technical review, code review, or compound learning.

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
# Find most recent plan comment on issue #42
gh api repos/{owner}/{repo}/issues/42/comments \
  --jq '[.[] | select(.body | contains("<!-- workflow-stage: plan"))] | last | .body'
```

Use `<!-- workflow-stage: {stage} -->` as the machine-readable marker. The emoji is for human readability; parsing uses the HTML comment.

## Stage Command Reference

These are compound-engineering plugin commands. Naming conventions (underscores in `/technical_review`, kebab in `/deepen-plan`) are upstream — not controlled by this skill. Validate issue number `#N` before use: digits only.

For error handling on all commands, see the [Error Handling](#error-handling) section.

| Command | Input | Output | Posts to | Stage / Emoji |
|---------|-------|--------|----------|---------------|
| `/workflows:brainstorm #N` | Issue body | Approaches, decisions, trade-offs | Issue (comment) | `brainstorm` 🔍 |
| `/workflows:plan #N` | Issue body + brainstorm | Implementation plan with file list | Issue (comment) | `plan` 📋 |
| `/deepen-plan` | Most recent plan comment | Enhanced plan with parallel research | Issue (comment, summary includes "Deepened") | `plan` 📋 |
| `/technical_review #N` | Most recent plan comment | Review from DHH, Kieran, Simplicity agents | Issue (comment) | `technical-review` 🔎 |
| `/workflows:work #N` | Most recent plan comment | Code on new branch, draft PR | Creates branch + draft PR | — |
| `/workflows:review` | Current PR | Multi-agent code review | PR (comments) | — |
| `/resolve_pr_parallel` | PR review comments | Auto-resolved simple fixes | Commits to PR branch | — |
| `/triage` | PR review comments | Interactive walkthrough per finding | User decides per-finding | — |
| `kdaws:security-review` | PR diff + codebase | Security scan with AI triage | PR (comment) | `security-review` 🛡️ |
| `/workflows:compound #N` | Issue thread + PR diff | Lesson in `docs/solutions/` | Issue (comment) | `compound` 📚 |

## Security Review: `kdaws:security-review`

Run a security scan with AI-powered triage. Can be invoked standalone or as part of a pipeline. Operates on the current branch/PR.

### Platform Detection

Auto-detect platform from project files on each run:

| Signal | Platform |
|--------|----------|
| `artisan` + `composer.json` with `laravel/framework` | Laravel |
| `wp-config.php` or `composer.json` with WordPress deps | WordPress |
| `go.mod` | Go |
| `composer.json` (no Laravel/WP signals) | PHP |
| No signals detected | Fallback — run semgrep with `p/ci` only, warn that platform-specific rules are not configured |

Multi-platform: detect ALL platforms present, run tools for each. Confirm with user on first detection.

### Tool Availability & Installation

Check required tools per platform. If missing, ask permission to install:

| Tool | Install | Platforms |
|------|---------|-----------|
| `semgrep` | `pip install semgrep` | All |
| `phpstan` | `composer require --dev phpstan/phpstan` | PHP, Laravel, WP |
| `larastan` | `composer require --dev larastan/larastan` | Laravel |
| `phpstan-wordpress` | `composer require --dev szepeviktor/phpstan-wordpress` | WordPress |
| `gosec` | `go install github.com/securego/gosec/v2/cmd/gosec@latest` | Go |
| `govulncheck` | `go install golang.org/x/vuln/cmd/govulncheck@latest` | Go |

> **Note:** `gitleaks` is a pre-commit tool installed by `kdaws:wf-setup`, not a runtime dependency of `kdaws:security-review`.

### Scan Steps

1. **Run scans** (parallel where possible, semgrep scoped to PR diff via `--baseline-commit`; other tools scan full codebase):

   PHP/Laravel/WordPress:
   ```bash
   semgrep scan --disable-nosem --baseline-commit $(git merge-base HEAD main) --config p/ci --config p/php [--config p/laravel] [--config p/phpcs-security-audit] --json
   phpstan analyse --level 8 --no-progress --error-format=json
   composer audit --format=json
   ```

   Go:
   ```bash
   semgrep scan --disable-nosem --baseline-commit $(git merge-base HEAD main) --config p/ci --config p/golang --json
   govulncheck ./...
   gosec -fmt=json ./...
   ```

2. **AI triage** — For each finding, Claude evaluates with full code context:
   - Real vulnerability → **block** (add to findings list)
   - Noise/false positive → **dismiss** (add justified `// nosemgrep: {rule-id} — {reason}` comment as a deliberate commit)
   - Dependency CVE → assess if vulnerable code path is actually used. Exploitable → block. Theoretical → warn.

3. **Post results to PR** — New collapsible comment each scan cycle (preserves audit trail):

   ```markdown
   <!-- workflow-stage: security-review, date: {YYYY-MM-DD}, command: kdaws:security-review -->
   <details>
   <summary>🛡️ Security Review — {YYYY-MM-DD} — {PASS|FAIL}</summary>

   **Platform:** {platforms} · **Tools:** {tools run}
   **Findings:** {N blocking}, {N dismissed}, {N warnings}

   ### Blocking Findings
   | Tool | Rule | Location | Description | Fix |
   |------|------|----------|-------------|-----|
   | ... | ... | ... | ... | ... |

   **Dismissed:** {N} false positives (nosemgrep comments added with justification)
   **Dependencies:** {N} CVEs triaged ({N} exploitable, {N} theoretical)

   ---
   *Generated by `kdaws:security-review` • Cycle {N}/3*
   </details>
   ```

4. **Block or proceed:**
   - Blocking findings exist → fix directly on the current branch, then re-run `kdaws:security-review`.
   - All clear → signal ready to proceed.

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

## Error Handling

- **GitHub API failures:** Fail immediately with the error message. User retries manually.
- **Auth failures:** If `gh` returns auth errors, prompt: "Run `gh auth login` to authenticate."
- **Missing issue:** If `gh issue view N` fails, error: "Issue #N not found. Check the issue number."
- **Branch conflicts:** If branch already exists, error: "Branch `feat/42-title` already exists. Delete it first or choose a different name."

## Rework and Rejection Paths

| Situation | Action |
|-----------|--------|
| Brainstorm reveals issue should be split | Close original issue. Create child issues with "Split from #N" in body. |
| Technical review identifies wrong approach | Loop back to plan (re-run `/workflows:plan`). Post revised plan as new collapsible comment. |
| Code review rejects PR entirely | Close the PR. Post a comment explaining why. Loop back to plan stage. |
| Review finds simple fixes | Run `/resolve_pr_parallel` then re-run `/workflows:review`. Max 2 review cycles. |
| Review finds complex fixes | Run `/triage` to walk through each with user. Then one final review pass. |
| Security review finds issues | Fix on current branch, re-run `kdaws:security-review`. Max 3 cycles, then escalate to manual review. |

## Review Cycle Limits

Maximum 2 review cycles per PR. After the second `/workflows:review`, prompt:

```
This PR has been reviewed twice. Options:
  1. Merge as-is
  2. Close PR and rework from plan stage
  3. Run one more review (not recommended)
```

This is a guideline enforced conversationally, not mechanically.

## Security Review Cycle Limits

Maximum 3 security review cycles per PR. After the third `kdaws:security-review`:

```
Security review has failed 3 times. Remaining findings need manual review:
  1. [finding]
  2. [finding]
Fix these manually or confirm as false positives before merging.
```

This is escalation to manual review, not approval to merge with known issues.
