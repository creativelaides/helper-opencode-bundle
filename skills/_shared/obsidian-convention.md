# Obsidian Artifact Convention (reference documentation)

NOTE: Critical Obsidian commands (`obsidian create`, `obsidian search`, `obsidian read`, `obsidian daily:append`) are inlined directly in each skill's SKILL.md. This document is supplementary reference — sub-agents do NOT need to read it to function.

## Overview

Obsidian is a local-first Markdown knowledge manager. When the persistence mode is `obsidian`, SDD artifacts live as notes in an Obsidian vault — navigable, linkable, searchable, and visualizable via graph view.

## Requirements

- **Obsidian desktop app** running (required for CLI and URI)
- **Obsidian CLI** enabled in Settings → General → Command line interface (requires Obsidian v1.8+)
- **A vault** that contains (or is) the project directory

If Obsidian CLI is not available, fall back to **direct filesystem** reads/writes on the vault directory — artifacts are plain Markdown files.

## Vault Structure

Point the Obsidian vault at the project root or a subdirectory. SDD artifacts live alongside your code:

```
{vault-root}/
├── .obsidian/                    ← Obsidian config (auto-managed)
├── openspec/                     ← OpenSpec artifacts (when openspec mode is also active)
│   ├── specs/
│   ├── changes/
│   └── ...
├── sdd/                          ← SDD artifacts (obsidian mode)
│   ├── {change-name}/
│   │   ├── proposal.md
│   │   ├── spec.md
│   │   ├── design.md
│   │   ├── tasks.md
│   │   ├── apply-progress.md
│   │   ├── verify-report.md
│   │   └── archive-report.md
│   └── init/
│       └── {project-name}.md
├── daily-notes/                  ← Daily notes (optional, for apply-progress)
│   └── 2026-04-21.md
└── templates/                    ← Obsidian templates (optional)
    ├── sdd-proposal.md
    ├── sdd-spec.md
    └── sdd-design.md
```

## Artifact Frontmatter

Every SDD artifact note includes YAML frontmatter for Obsidian's metadata cache and Dataview queries:

```yaml
---
title: sdd/{change-name}/{artifact-type}
topic_key: sdd/{change-name}/{artifact-type}
type: architecture
project: {project-name}
change: {change-name}
artifact: {artifact-type}
tags: [sdd, {artifact-type}, {change-name}]
status: draft           # draft | in-review | approved | implemented | archived
created: {ISO date}
updated: {ISO date}
---
```

This enables Dataview queries like:
```dataview
TABLE status, updated
FROM "sdd"
WHERE change = "add-dark-mode"
SORT artifact ASC
```

## Inter-Market Linking

Obsidian wikilinks create the dependency graph automatically. Include backlinks in each artifact:

```markdown
## Dependencies
- [[sdd/{change-name}/proposal|Proposal]]
- [[sdd/{change-name}/spec|Spec]]

## Unlocks
- [[sdd/{change-name}/tasks|Tasks]] (after this artifact is done)
```

## Interfaces

There are three interfaces, in priority order:

### 1. Obsidian CLI (preferred)

Requires Obsidian desktop running. Most capable interface.

| Action | Command |
|--------|---------|
| Create note | `obsidian create name="sdd/{change-name}/proposal" content="{markdown}" folder="sdd/{change-name}"` |
| Read note | `obsidian read file="sdd/{change-name}/proposal"` |
| Search vault | `obsidian search query="sdd/{change-name}" format=json` |
| Append to daily note | `obsidian daily:append content="- [x] Completed task 1.1 in {change-name}"` |
| List tasks | `obsidian tasks file="sdd/{change-name}/tasks"` |
| Open note in app | `obsidian open file="sdd/{change-name}/design"` |
| Tags with counts | `obsidian tags counts` |

**Detection**: `obsidian --version` exits 0 → CLI available.

### 2. Obsidian URI (fallback — no CLI)

Works on any system with Obsidian installed. Limited to open/create/search.

| Action | URI |
|--------|-----|
| Open note | `obsidian://open?vault={vault}&file=sdd/{change-name}/proposal` |
| Create note | `obsidian://new?vault={vault}&name=sdd/{change-name}/proposal&content={url-encoded-markdown}` |
| Search | `obsidian://search?vault={vault}&query=sdd/{change-name}` |

**Detection**: Test if `obsidian://` URI scheme is registered (platform-dependent).

### 3. Direct Filesystem (always available)

Read/write Markdown files directly in the vault directory. No Obsidian app required.

| Action | Method |
|--------|--------|
| Create note | Write file at `{vault-root}/sdd/{change-name}/{artifact}.md` |
| Read note | Read file at `{vault-root}/sdd/{change-name}/{artifact}.md` |
| Search | Grep across `sdd/` directory |
| List notes | Glob `sdd/**/*.md` |

**Detection**: Always available. Use when CLI and URI are unavailable.

## Naming Rules

Identical to the memory-convention naming, adapted for filesystem paths:

```
Path:      sdd/{change-name}/{artifact-type}.md
Title:     sdd/{change-name}/{artifact-type}
topic_key: sdd/{change-name}/{artifact-type}
type:      architecture
project:   {detected or current project name}
tags:      [sdd, {artifact-type}, {change-name}]
```

### Artifact Types

| Artifact Type | File | Produced By |
|---------------|------|-------------|
| `explore` | `sdd/{change-name}/explore.md` | sdd-explore |
| `proposal` | `sdd/{change-name}/proposal.md` | sdd-propose |
| `spec` | `sdd/{change-name}/spec.md` | sdd-spec |
| `design` | `sdd/{change-name}/design.md` | sdd-design |
| `tasks` | `sdd/{change-name}/tasks.md` | sdd-tasks |
| `apply-progress` | `sdd/{change-name}/apply-progress.md` | sdd-apply |
| `verify-report` | `sdd/{change-name}/verify-report.md` | sdd-verify |
| `archive-report` | `sdd/{change-name}/archive-report.md` | sdd-archive |
| `state` | `sdd/{change-name}/state.md` | orchestrator |

Exception: `sdd-init` uses `sdd/init/{project-name}.md`.

## Writing Artifacts

With CLI:
```bash
obsidian create name="sdd/{change-name}/proposal" folder="sdd/{change-name}" content="{frontmatter + markdown}"
```

With filesystem:
```bash
# Write to vault directory
cat > "{vault-root}/sdd/{change-name}/proposal.md" << 'FRONTMATTER'
---
title: sdd/{change-name}/proposal
topic_key: sdd/{change-name}/proposal
type: architecture
project: "{project}"
change: "{change-name}"
artifact: proposal
tags: [sdd, proposal, {change-name}]
status: draft
created: {ISO date}
updated: {ISO date}
---
FRONTMATTER
# then append content
```

Update existing artifact: rewrite the entire file ( Obsidian detects changes on disk).

## Reading Artifacts

With CLI:
```bash
obsidian read file="sdd/{change-name}/proposal"
```

With filesystem:
```bash
cat "{vault-root}/sdd/{change-name}/proposal.md"
```

## Daily Notes Integration

sdd-apply can log progress to daily notes for chronological tracking:

```bash
obsidian daily:append content="- [x] sdd/{change-name}: Completed task 1.1 - Create auth middleware"
obsidian daily:append content="- [ ] sdd/{change-name}: Next - Add auth routes (task 1.3)"
```

Daily notes become an activity log that Obsidian's calendar plugin visualizes.

## Task Checkbox Integration

Obsidian Tasks plugin reads `- [ ]` and `- [x]` checkboxes. The tasks.md format is already compatible:

```markdown
## Phase 1: Foundation

- [x] 1.1 Create `internal/auth/middleware.go`     ← Obsidian Tasks sees this
- [ ] 1.2 Add `AuthConfig` struct                 ← And this
```

Query via CLI:
```bash
obsidian tasks file="sdd/{change-name}/tasks"
```

## Graph View

Obsidian's graph view auto-generates from wikilinks. Include dependency links in each artifact:

```markdown
## Depends on
- [[sdd/{change-name}/proposal|← Proposal]]

## Required by
- [[sdd/{change-name}/tasks|Tasks →]]
```

The graph view then shows the full artifact dependency chain as a navigable graph.

## Recovery Protocol

Same two-step pattern as memory convention:

```
Step 1: Search — obsidian search query="sdd/{change-name}/" format=json
Step 2: Read   — obsidian read file="sdd/{change-name}/{artifact}"
```

Or via filesystem:
```
Step 1: glob sdd/{change-name}/*.md
Step 2: read each file
```

## Coexistence with OpenSpec

When both `obsidian` and `openspec` modes are active (hybrid), artifacts can live in the `openspec/` directory which the Obsidian vault also indexes. The `sdd/` and `openspec/` directories can coexist — use wikilinks across them:

```markdown
// In openspec/changes/add-dark-mode/proposal.md:
Related: [[sdd/add-dark-mode/proposal|SDD Proposal]]

// In sdd/add-dark-mode/proposal.md:
Source: [[openspec/changes/add-dark-mode/proposal|OpenSpec Proposal]]
```

## Why This Convention

- Obsidian makes SDD artifacts **navigable** — click through proposals → specs → design → tasks via links
- Graph view gives **visual dependency maps** of changes
- Daily notes create **chronological activity logs** across sessions
- Tasks plugin gives **native checkbox tracking** with queries and filters
- Local-first: vault is **just Markdown files** — works offline, no server, git-friendly
- Frontmatter enables **Dataview queries** for dashboards and status boards
- Coexists with memory (API) and openspec (CLI) backends in hybrid mode