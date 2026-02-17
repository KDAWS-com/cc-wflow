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
