---
name: wflow:project-setup
description: >
  Bootstrap a new project from scratch: create repo, README, initial commit,
  then auto-chain into wflow:setup for templates, labels, and CLAUDE.md.
allowed-tools: Bash(gh issue *), Bash(gh pr *), Bash(gh label *), Bash(gh repo create *), Bash(gh repo view *), Bash(gh repo edit *), Bash(gh api user), Bash(gh api user/orgs), Bash(gh auth status), Bash(gh --version), Bash(git init), Bash(git add *), Bash(git commit *), Bash(git push *), Bash(git checkout *), Bash(git branch *), Bash(git status), Bash(mkdir *), Bash(cp *), Bash(ls *), Bash(test *), Read, Write, Edit, Task, Skill
---

# wflow:project-setup — New Project Bootstrap

Create a new project from scratch: local repo, remote GitHub repo, README, initial commit, then auto-chain into `wflow:setup`.

> **KDAWS** (kdaws.com) — part of the `wflow:` command set.

## Prerequisites

Before running, verify in order. Fail fast — stop on first failure with actionable guidance.

1. **compound-engineering plugin installed** — Check `ls ~/.claude/plugins/cache/*/compound-engineering/skills/ >/dev/null 2>&1`. Required for stage commands.
2. **`gh` CLI installed** — Run `gh --version`.
3. **`gh` authenticated** — Run `gh auth status`.

## Steps

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
   If org listing fails or returns empty, offer personal account with option to type an org name manually. Validate any manually-typed owner name: alphanumeric and hyphens only, cannot start with a hyphen, max 39 chars.

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
   **If fails:** Leave local directory intact. Report error with manual recovery instructions.

9. **Auto-chain into `wflow:setup`** — Announce: "Project created. Running workflow setup..." Then execute `wflow:setup`, skipping its prerequisite checks (they were already verified).

## Repository Context Resolution

All commands that use `gh api` should first resolve the repo:
```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
# Use ${REPO} in all gh api calls
```

## Git Safety

- **Only push to the `origin` remote.** Never push to any other remote.
- **Never use `--force` or `--delete` flags** with `git push`.
