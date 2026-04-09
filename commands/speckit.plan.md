<!-- Based on spec-kit v0.5.1 (SHA: aa2282e) — core content from github/spec-kit -->
---
description: Execute the implementation planning workflow using the plan template to generate design artifacts.
handoffs: 
  - label: Create Tasks
    agent: speckit.tasks
    prompt: Break the plan into tasks
    send: true
  - label: Create Checklist
    agent: speckit.checklist
    prompt: Create a checklist for the following domain...
scripts:
  sh: scripts/bash/setup-plan.sh --json
  ps: scripts/powershell/setup-plan.ps1 -Json
agent_scripts:
  sh: scripts/bash/update-agent-context.sh __AGENT__
  ps: scripts/powershell/update-agent-context.ps1 -AgentType __AGENT__
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before planning)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_plan` key
- If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally
- Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
- For each executable hook, output the following based on its `optional` flag:
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```
  - **Mandatory hook** (`optional: false`):
    ```
    ## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Outline.
    ```
- If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently

## Outline

1. **Setup**: Run `{SCRIPT}` from repo root and parse JSON for FEATURE_SPEC, IMPL_PLAN, SPECS_DIR, BRANCH. For single quotes in args like "I'm Groot", use escape syntax: e.g 'I'\''m Groot' (or double-quote if possible: "I'm Groot").

2. **Load context**: Read FEATURE_SPEC and `/memory/constitution.md`. Load IMPL_PLAN template (already copied).

<!-- PRESET: multi-repo-branching START -->
3. **Discover child repositories**:
   Read `.specify/init-options.json` from the repo root. Look for the `multi_repo_branching` object:
   ```json
   {
     "multi_repo_branching": {
       "scan_depth": 3,
       "type": "auto"
     }
   }
   ```
   - `type`: `"auto"` (default), `"independent"`, or `"submodule"`
   - `scan_depth`: max directory depth to scan (default: 2)
   - If `init-options.json` does not exist or has no `multi_repo_branching` key, use defaults: `type = "auto"`, `scan_depth = 2`

   **Based on type, discover repos:**

   - **If type is `"submodule"`**: Parse `.gitmodules` in the repo root to extract submodule paths. For each entry, the `path` value is the child repo location.

   - **If type is `"independent"`**: Run a shell command to find directories containing `.git` that are NOT the repo root:
     - Bash: `find . -maxdepth <scan_depth> -name .git -type d 2>/dev/null | sed 's|/\.git$||' | grep -v '^\.$' | sort`
     - PowerShell: `Get-ChildItem -Path . -Filter .git -Directory -Recurse -Depth <scan_depth> -Force | Where-Object { $_.Parent.FullName -ne (Get-Location).Path } | ForEach-Object { $_.Parent.FullName } | Sort-Object`
     - For each discovered path, check if it is gitignored by the parent repo: run `git check-ignore -q <path>`. If the path IS ignored but contains `.git`, it is still a valid child repo (include it). If a parent directory is ignored and does NOT contain `.git`, skip it (do not descend).

   - **If type is `"auto"`**: Check if `.gitmodules` exists in the repo root.
     - If yes → use `"submodule"` discovery
     - If no → use `"independent"` discovery

   If no child repos are discovered, skip the repo analysis step and continue normally.

4. **Identify affected repositories**: If child repos were discovered:
   - Read the feature spec (FEATURE_SPEC)
   - For each discovered child repo, determine whether this feature requires changes in that repo based on the spec's requirements, user stories, and technical scope
   - Add an **Affected Repositories** section under **Project Structure** in plan.md with a table:

     ```markdown
     ### Affected Repositories

     | Repo Path | Type | Reason |
     |-----------|------|--------|
     | components/auth | independent | New OAuth2 provider needs auth module changes |
     | libs/shared | submodule | Shared types needed for the new API contract |
     ```

   - This information will be used by `/speckit.tasks` to generate a setup task for creating feature branches in the affected repos
   - If no repos are affected, do not add the section
<!-- PRESET: multi-repo-branching END -->

5. **Execute plan workflow**: Follow the structure in IMPL_PLAN template to:
   - Fill Technical Context (mark unknowns as "NEEDS CLARIFICATION")
   - Fill Constitution Check section from constitution
   - Evaluate gates (ERROR if violations unjustified)
   - Phase 0: Generate research.md (resolve all NEEDS CLARIFICATION)
   - Phase 1: Generate data-model.md, contracts/, quickstart.md
   - Phase 1: Update agent context by running the agent script
   - Re-evaluate Constitution Check post-design

6. **Stop and report**: Command ends after Phase 2 planning. Report branch, IMPL_PLAN path, generated artifacts, and affected repos (if any).

7. **Check for extension hooks**: After reporting, check if `.specify/extensions.yml` exists in the project root.
   - If it exists, read it and look for entries under the `hooks.after_plan` key
   - If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally
   - Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
   - For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
     - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
     - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
   - For each executable hook, output the following based on its `optional` flag:
     - **Optional hook** (`optional: true`):
       ```
       ## Extension Hooks

       **Optional Hook**: {extension}
       Command: `/{command}`
       Description: {description}

       Prompt: {prompt}
       To execute: `/{command}`
       ```
     - **Mandatory hook** (`optional: false`):
       ```
       ## Extension Hooks

       **Automatic Hook**: {extension}
       Executing: `/{command}`
       EXECUTE_COMMAND: {command}
       ```
   - If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently

## Phases

### Phase 0: Outline & Research

1. **Extract unknowns from Technical Context** above:
   - For each NEEDS CLARIFICATION → research task
   - For each dependency → best practices task
   - For each integration → patterns task

2. **Generate and dispatch research agents**:

   ```text
   For each unknown in Technical Context:
     Task: "Research {unknown} for {feature context}"
   For each technology choice:
     Task: "Find best practices for {tech} in {domain}"
   ```

3. **Consolidate findings** in `research.md` using format:
   - Decision: [what was chosen]
   - Rationale: [why chosen]
   - Alternatives considered: [what else evaluated]

**Output**: research.md with all NEEDS CLARIFICATION resolved

### Phase 1: Design & Contracts

**Prerequisites:** `research.md` complete

1. **Extract entities from feature spec** → `data-model.md`:
   - Entity name, fields, relationships
   - Validation rules from requirements
   - State transitions if applicable

2. **Define interface contracts** (if project has external interfaces) → `/contracts/`:
   - Identify what interfaces the project exposes to users or other systems
   - Document the contract format appropriate for the project type
   - Examples: public APIs for libraries, command schemas for CLI tools, endpoints for web services, grammars for parsers, UI contracts for applications
   - Skip if project is purely internal (build scripts, one-off tools, etc.)

3. **Agent context update**:
   - Run `{AGENT_SCRIPT}`
   - These scripts detect which AI agent is in use
   - Update the appropriate agent-specific context file
   - Add only new technology from current plan
   - Preserve manual additions between markers

**Output**: data-model.md, /contracts/*, quickstart.md, agent-specific file

## Key rules

- Use absolute paths
- ERROR on gate failures or unresolved clarifications
