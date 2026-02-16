---
title: Add security review stage to workflow processes
date: 2026-02-16
category: integration-issues
tags: [security, semgrep, skill-authoring, workflow-integration, code-review, platform-detection, ai-triage, allowed-tools]
severity: medium
component: SKILL.md
issue: "#3"
pr: "#4"
---

# Security Review Stage: Skill Integration Pattern

How we added a `kdaws:security-review` command to the GitHub workflow skill, and what we learned about skill authoring, tool permissions, and cross-reference consistency.

## Problem

We needed an automated security review stage integrated into both full and simple workflow pipelines. Previous experience with Snyk showed too much noise — it didn't understand code context. We needed high-confidence results using targeted tools with AI-powered triage.

The challenge was not just selecting the right tools, but designing the integration pattern: how the stage fits into the workflow, how tool permissions are scoped, how findings are managed, and how the skill definition stays internally consistent across multiple cross-referencing sections.

## Root Cause of Key Issues

The initial brainstorm and plan contained several over-engineered or over-permissive design decisions that the review process caught:

1. **Destructive file modification for scanning** — stripping nosemgrep comments before scanning
2. **Over-broad tool permissions** — `Bash(semgrep *)` instead of subcommand-scoped
3. **Unscoped scans** — reporting entire codebase findings on every PR
4. **Unnecessary state management** — caching platform detection in config files
5. **Over-engineered fix flows** — sub-issue workflows for mechanical fixes
6. **Cross-reference drift** — same information in 4+ locations becoming inconsistent

## Solution

### 1. Non-destructive semgrep ignore handling

Use `semgrep --disable-nosem` flag instead of stripping comments from files. Achieves the same goal (ignores inline suppressions during scanning) without modifying source files.

```bash
# Correct: flag-based, non-destructive
semgrep scan --disable-nosem --baseline-commit $(git merge-base HEAD main) --config p/ci --config p/php --json

# Wrong: destructive file modification with unclear restore semantics
# grep -rn 'nosemgrep' --include='*.php' | sed ... (strip comments, scan, restore)
```

When AI triage dismisses a finding as false positive, add a justified suppression as a deliberate commit:

```php
// nosemgrep: php.lang.security.injection.tainted-sql-string — False positive: $table is validated against allowlist in constructor
$query = "SELECT * FROM {$this->table} WHERE id = ?";
```

### 2. Least-privilege tool permissions

Restrict `allowed-tools` frontmatter to specific subcommands:

```yaml
# Correct: scoped to specific subcommands
allowed-tools: Bash(semgrep scan *), Bash(phpstan analyse *), Bash(composer audit *), Bash(govulncheck *), Bash(gosec *), Bash(gitleaks detect *), Bash(gitleaks protect *)

# Wrong: over-broad, allows semgrep login/publish/install etc.
allowed-tools: Bash(semgrep *), Bash(phpstan *), Bash(gitleaks *)
```

### 3. Scoping findings to PR diff

Use `--baseline-commit` to scope semgrep to changed code only. Note: PHPStan, govulncheck, and gosec lack native baseline support — they scan the full codebase.

### 4. Stateless platform detection

Auto-detect platform from project files on every run instead of caching in config:

| Signal | Platform |
|--------|----------|
| `artisan` + `composer.json` with `laravel/framework` | Laravel |
| `wp-config.php` or `composer.json` with WordPress deps | WordPress |
| `go.mod` | Go |
| `composer.json` (no Laravel/WP signals) | PHP |
| No signals detected | Fallback — semgrep with `p/ci` only |

### 5. Simplified fix workflow

Fix directly on current branch, re-run `kdaws:security-review`. Max 3 cycles, then escalate to manual review. No sub-issue workflows.

## Prevention Strategies

### Skill Authoring

- **Use tool flags over file modifications.** When tools support option flags (`--disable-nosem`, `--no-progress`), use them rather than pre-processing files.
- **Apply least privilege before review.** Audit what subcommands are actually used and restrict the `allowed-tools` allowlist accordingly.
- **Avoid caching for cheap recomputation.** Platform detection from project files is fast — don't introduce state management for it.
- **Simplify fix flows to minimum viable process.** Fix on branch, re-run scan. No sub-issues, no complex multi-step handoffs.

### Cross-Reference Consistency

- **Verify cross-references explicitly in review.** When the same information appears in multiple places (quick reference, CLAUDE.md template, detection prompt, orchestrator steps), review must check all locations match.
- **Pin tool versions explicitly.** Gitleaks `v8.30.0` in the SKILL.md template must match the committed `.pre-commit-config.yaml`. The review caught a version mismatch.
- **Document exception cases explicitly.** When a command doesn't require an issue number, list that exception everywhere the requirement is stated.

### Workflow Discipline

- **Post your plan to the issue before coding.** During this implementation, we started coding before posting the plan. The workflow exists for alignment — post plan first, then implement.
- **Run the workflow you're building.** If you're implementing a skill that defines a workflow, follow that workflow for the skill's own development.

## Related Documentation

- [Namespace Separation and Skill Consolidation](../skill-architecture/namespace-separation-and-skill-consolidation.md) — Prior architectural decisions from Issue #1/#2
- [Issue #3](https://github.com/KarlAustin/github-workflow-skill/issues/3) — Feature request with brainstorm and plan comments
- [PR #4](https://github.com/KarlAustin/github-workflow-skill/pull/4) — Implementation with 2-cycle review (14 findings, all resolved)
