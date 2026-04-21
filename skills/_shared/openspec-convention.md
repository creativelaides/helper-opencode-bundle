# OpenSpec File Convention (shared across all SDD skills)

## Directory Structure

```
openspec/
├── config.yaml              ← Project-specific SDD config
├── specs/                   ← Source of truth (main specs)
│   └── {domain}/
│       └── spec.md
└── changes/                 ← Active changes
    ├── archive/             ← Completed changes (YYYY-MM-DD-{change-name}/)
    └── {change-name}/       ← Active change folder
        ├── state.yaml       ← DAG state (survives compaction)
        ├── exploration.md   ← (optional) from sdd-explore
        ├── proposal.md      ← from sdd-propose
        ├── specs/           ← from sdd-spec
        │   └── {domain}/
        │       └── spec.md  ← Delta spec
        ├── design.md        ← from sdd-design
        ├── tasks.md         ← from sdd-tasks (updated by sdd-apply)
        └── verify-report.md ← from sdd-verify
```

## CLI Integration

OpenSpec provides a CLI (`openspec`) for structured operations. Use it when available:

| Action | Command | Output |
|--------|---------|--------|
| Check artifact status | `openspec status --change "{change-name}" --json` | Artifact states: blocked/ready/done |
| Get enriched instructions | `openspec instructions {artifact-id} --change "{change-name}" --json` | Template + deps + context + rules |
| List changes | `openspec list` | Active changes |
| Archive change | `openspec archive "{change-name}"` | Moves to archive, syncs specs |
| Validate | `openspec validate "{change-name}"` | Validation results |
| View dashboard | `openspec view` | Interactive dashboard |
| List specs | `openspec list --specs` | All main specs |

### Fallback: Manual File Operations

If `openspec` CLI is not available, read/write files directly per the paths below.

## Artifact File Paths

| Skill | Creates / Reads | Path |
|-------|----------------|------|
| orchestrator | Creates/Updates | `openspec/changes/{change-name}/state.yaml` |
| sdd-init | Creates | `openspec/config.yaml`, `openspec/specs/`, `openspec/changes/`, `openspec/changes/archive/` |
| sdd-explore | Creates (optional) | `openspec/changes/{change-name}/exploration.md` |
| sdd-propose | Creates | `openspec/changes/{change-name}/proposal.md` |
| sdd-spec | Creates | `openspec/changes/{change-name}/specs/{domain}/spec.md` |
| sdd-design | Creates | `openspec/changes/{change-name}/design.md` |
| sdd-tasks | Creates | `openspec/changes/{change-name}/tasks.md` |
| sdd-apply | Updates | `openspec/changes/{change-name}/tasks.md` (marks `[x]`) |
| sdd-verify | Creates | `openspec/changes/{change-name}/verify-report.md` |
| sdd-archive | Moves | `openspec/changes/{change-name}/` → `openspec/changes/archive/YYYY-MM-DD-{change-name}/` |
| sdd-archive | Updates | `openspec/specs/{domain}/spec.md` (merges deltas into main specs) |

## Reading Artifacts

```
Proposal:   openspec/changes/{change-name}/proposal.md
Specs:      openspec/changes/{change-name}/specs/  (all domain subdirectories)
Design:     openspec/changes/{change-name}/design.md
Tasks:      openspec/changes/{change-name}/tasks.md
Verify:     openspec/changes/{change-name}/verify-report.md
Config:     openspec/config.yaml
Main specs: openspec/specs/{domain}/spec.md
```

## Writing Rules

- Always create the change directory before writing artifacts
- If a file already exists, READ it first and UPDATE it (don't overwrite blindly)
- If the change directory already exists with artifacts, the change is being CONTINUED
- Use `openspec/config.yaml` `rules` section for project-specific constraints per phase
- When `openspec` CLI is available, prefer CLI for status checks and enriched instructions over manual file reads

## Config File Reference

```yaml
# openspec/config.yaml
schema: spec-driven

context: |
  Tech stack: {detected}
  Architecture: {detected}
  Testing: {detected}
  Style: {detected}

rules:
  proposal:
    - Include rollback plan for risky changes
  specs:
    - Use Given/When/Then for scenarios
    - Use RFC 2119 keywords (MUST, SHALL, SHOULD, MAY)
  design:
    - Include sequence diagrams for complex flows
    - Document architecture decisions with rationale
  tasks:
    - Group by phase, use hierarchical numbering
    - Keep tasks completable in one session
  apply:
    - Follow existing code patterns
    tdd: false           # Set to true to enable RED-GREEN-REFACTOR
    test_command: ""
  verify:
    test_command: ""
    build_command: ""
    coverage_threshold: 0
  archive:
    - Warn before merging destructive deltas
```

## Archive Structure

When archiving, the change folder moves to:
```
openspec/changes/archive/YYYY-MM-DD-{change-name}/
```

Use today's date in ISO format. The archive is an AUDIT TRAIL — never delete or modify archived changes.

## OPSX Slash Commands

OpenSpec ships skills that register slash commands. These are available after running `openspec init --tools opencode` and `openspec update` in the project:

| Command | Action |
|---------|--------|
| `/opsx:propose` | Create a change and generate planning artifacts |
| `/opsx:explore` | Think through ideas, investigate codebase |
| `/opsx:apply` | Implement tasks |
| `/opsx:archive` | Archive when done |

These commands coexist with SDD sub-agents — the SDD pipeline provides structured artifact persistence via the persistence contract, while OPSX commands provide the interactive fluency.