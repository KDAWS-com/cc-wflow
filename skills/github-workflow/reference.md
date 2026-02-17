# Reference

Lookup tables, error handling, rework paths, and cross-plugin contracts. Load this file when handling errors or rework situations.

## Stage Command Reference

These are compound-engineering plugin commands. Naming conventions (underscores in `/technical_review`, kebab in `/deepen-plan`) are upstream — not controlled by this skill. Validate issue number `#N` before use: digits only.

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
| `quality:security-review` | PR diff + codebase | Security scan with AI triage | PR (comment) | `security-review` 🛡️ |
| `/workflows:compound #N` | Issue thread + PR diff | Lesson in `docs/solutions/` | Issue (comment) | `compound` 📚 |

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
| Security review finds issues | Fix on current branch, re-run `quality:security-review`. Max 3 cycles, then escalate to manual review. |

## Cross-Plugin Interface Contract: quality:security-review

The `wflow` orchestrator expects the following from `quality:security-review`:

- **Invocation:** Use the Skill tool: `skill: "quality:security-review"` (no arguments needed).
- **Input:** Runs on current branch/PR context. No arguments required.
- **Output:** Posts a collapsible PR comment with `<!-- workflow-stage: security-review -->` marker.
- **Return signal:** Comment summary line contains `PASS` or `FAIL`.
- **Cycle ownership:** The wflow orchestrator owns the 3-cycle limit. It counts invocations and escalates after the 3rd failure. The quality plugin does not track cycles.
- **Install command:** `/plugin install quality@kdaws`
- **Minimum version:** 1.0.0 (version checking is not enforced at runtime; both plugins are co-maintained).
