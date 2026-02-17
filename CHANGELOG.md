# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-02-17

### Added
- Initial plugin release as `wflow`
- `wflow:full` — complete pipeline orchestrator (brainstorm → plan → review → work → security review → compound)
- `wflow:simple` — quick pipeline orchestrator (plan → work → security review)
- `wflow:setup` — project workflow setup (templates, labels, CLAUDE.md section)
- `wflow:project-setup` — new project bootstrap
- Detection and prompting for code-creation intent
- Progressive disclosure skill structure (SKILL.md + orchestrators.md + reference.md)
- Cross-plugin interface contract for `quality:security-review`

