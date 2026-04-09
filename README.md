# Multi-Repo Branching for Spec Kit

A community preset for [Spec Kit](https://github.com/github/spec-kit) that coordinates feature branch creation across multiple git repositories during the **plan** and **tasks** phases of the spec-driven development workflow.

## Problem

In multi-module projects where sub-components maintain independent git histories (stored within a shared root) or use git submodules, Spec Kit only operates on the root repository. Developers must manually create matching feature branches in each affected sub-component.

## What This Preset Does

| Phase | Without Preset | With Preset |
|-------|---------------|-------------|
| **Plan** | Analyzes root repo only | Discovers child repos, identifies which are affected by the feature |
| **Tasks** | No branch setup tasks | Generates Phase 1 tasks to create feature branches in affected repos |

The preset supports two types of child repositories:

- **Independent repos** — subdirectories with their own `.git` (not formal git submodules)
- **Git submodules** — formally registered in `.gitmodules`

## Installation

```bash
# Install from GitHub release
specify preset add --from https://github.com/sakitA/spec-kit-preset-multi-repo-branching/archive/refs/tags/v1.0.0.zip

# Or install from local directory (for development)
specify preset add --dev /path/to/spec-kit-preset-multi-repo-branching

# Verify installation
specify preset list
specify preset resolve speckit.plan
```

## Configuration

Add to `.specify/init-options.json` in your project root:

```json
{
  "multi_repo_branching": {
    "scan_depth": 3,
    "type": "independent"
  }
}
```

### Options

| Option | Values | Default | Description |
|--------|--------|---------|-------------|
| `type` | `"independent"`, `"submodule"`, `"auto"` | `"auto"` | How to discover child repos |
| `scan_depth` | `1`–`10` | `2` | Max directory depth to scan |

### Type Modes

- **`independent`** — Scans for directories containing `.git` that are NOT the repo root. Uses `git check-ignore` to filter gitignored directories during traversal. A directory with its own `.git` is always reported even if gitignored.
- **`submodule`** — Parses `.gitmodules` in the repo root to find registered submodule paths. No filesystem scanning needed.
- **`auto`** (default) — Checks if `.gitmodules` exists. If yes, uses submodule mode. If no, uses independent mode.

### Configuration Not Required

If `.specify/init-options.json` doesn't exist or has no `multi_repo_branching` key, the preset uses defaults: `type = "auto"`, `scan_depth = 2`.

## How It Works

### During `/speckit.plan`

1. After running the setup script, the AI agent reads your config
2. Based on the `type` setting:
   - **Independent**: Runs `find` to locate `.git` directories, filters with `git check-ignore`
   - **Submodule**: Parses `.gitmodules` for submodule paths
   - **Auto**: Checks for `.gitmodules`, falls back to independent scan
3. Cross-references discovered repos against the feature spec
4. Documents affected repos in the plan's **Affected Repositories** section

### During `/speckit.tasks`

1. Reads the "Affected Repositories" section from `plan.md`
2. Generates Phase 1 setup tasks for branch creation:
   - **Independent**: `git -C "<path>" checkout -b "<BRANCH>"`
   - **Submodule**: `git submodule update --init "<path>" && git -C "<path>" checkout -b "<BRANCH>"`

## Examples

### Independent Repos (Monorepo Without Submodules)

Project structure:
```
my-project/           # Root repo (.git)
├── components/
│   ├── auth/         # Independent repo (.git)
│   └── api/          # Independent repo (.git)
├── libs/
│   └── shared/       # Independent repo (.git)
└── .specify/
    └── init-options.json
```

Config:
```json
{
  "multi_repo_branching": {
    "type": "independent",
    "scan_depth": 3
  }
}
```

### Git Submodules

Project structure:
```
my-project/           # Root repo (.git)
├── vendor/
│   ├── lib-a/        # Submodule
│   └── lib-b/        # Submodule
├── .gitmodules
└── .specify/
    └── init-options.json
```

Config:
```json
{
  "multi_repo_branching": {
    "type": "submodule"
  }
}
```

### Auto-Detection (No Config Needed)

If you don't add any config, the preset auto-detects:
- Checks for `.gitmodules` → submodule mode
- No `.gitmodules` → independent scan with depth 2

## What's Overridden

This preset overrides **commands only** — no template overrides. Core templates are used as-is, and the commands instruct the AI agent to add multi-repo sections when generating plan.md and tasks.md.

| File | Type | Core Behavior Preserved | Additions |
|------|------|------------------------|-----------|
| `speckit.plan` | command | ✅ Full workflow | + Steps 3-4: discover repos, identify affected, add section to plan.md |
| `speckit.tasks` | command | ✅ Full workflow | + Branch-creation task generation in Phase 1 |

### Why commands-only?

Overriding templates requires copying the entire core template to add a few lines. When upstream updates templates, presets become stale. By keeping template overrides out, the AI uses the latest core templates and adds multi-repo sections dynamically via command instructions.

Command overrides are clearly marked with `<!-- PRESET: multi-repo-branching START/END -->` comments around additions for easy diffing when upstream changes.

### Composition readiness

Spec Kit has a [composition strategies feature](https://github.com/github/spec-kit) in development (`append`, `prepend`, `wrap`). When it lands, this preset can migrate to `strategy: wrap` for commands — eliminating all core content duplication.

## Stacking

This preset is designed to work well when stacked with other presets. It only adds sections to templates and steps to commands — it does not remove or reorder any core behavior.

If another preset also overrides `speckit.plan` or `speckit.tasks`, the one with higher priority wins. Use `--priority` when installing to control order.

## Removing

```bash
specify preset remove multi-repo-branching
```

## License

MIT
