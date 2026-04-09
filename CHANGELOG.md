# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-04-09

### Added

- Plan command override with multi-repo discovery
  - Auto-scan for independent `.git` directories
  - Git submodule detection via `.gitmodules` parsing
  - Auto mode that detects which type applies
  - Configurable scan depth
  - `.gitignore`-aware directory filtering
- Tasks command override with branch-creation task generation
  - Type-aware branching: `git -C` for independent, `git submodule update` for submodules
  - Generated as Phase 1 setup tasks
- Commands instruct AI to add "Affected Repositories" section to plan.md dynamically
- Commands instruct AI to add Phase 1 branch-creation tasks to tasks.md dynamically
- Configuration via `.specify/init-options.json`
- Upstream version tracking via `<!-- Based on spec-kit vX.Y.Z -->` comment in commands
- Clear `<!-- PRESET: multi-repo-branching START/END -->` markers around all preset additions
