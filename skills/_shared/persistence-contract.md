# Persistence Contract (shared across all SDD skills)

## Mode Resolution

The orchestrator passes `artifact_store.mode` with one of: `memory | obsidian | openspec | hybrid | none`.

Default resolution (when orchestrator does not explicitly set a mode):
1. If Memory service is available (http://127.0.0.1:8192 returns healthy) → use `memory`
2. If Obsidian CLI is available (`obsidian --version` exits 0) → use `obsidian`
3. Otherwise → use `none`

`openspec` and `hybrid` are NEVER used by default — only when explicitly passed.

When falling back to `none`, recommend the user start the memory server (`memory launch`), enable Obsidian, or enable `openspec`.

## Behavior Per Mode

| Mode | Read from | Write to | Project files |
|------|-----------|----------|---------------|
| `memory` | Memory HTTP API | Memory HTTP API | Never |
| `obsidian` | Obsidian CLI / URI / Filesystem | Obsidian vault (Markdown files) | Yes (`sdd/` dir) |
| `openspec` | OpenSpec CLI / Filesystem | Filesystem | Yes (`openspec/` dir) |
| `hybrid` | Memory (primary) + Filesystem (fallback) | Both | Yes |
| `none` | Orchestrator prompt context | Nowhere | Never |

### Obsidian Mode

Artifacts live as Markdown notes in the Obsidian vault. Benefits:
- **Navigable** — click through linked artifacts in Obsidian
- **Graph view** — visualize artifact dependencies as a graph
- **Daily notes** — chronological progress log via `obsidian daily:append`
- **Task tracking** — native `- [ ]` / `- [x]` checkbox support via Tasks plugin
- **Dataview queries** — frontmatter enables dashboards and status boards
- **Local-first** — just Markdown files, works offline, git-compatible

Full convention: `skills/_shared/obsidian-convention.md`.

### Hybrid Mode

Persists every artifact to BOTH Memory and another backend simultaneously:
- Memory: cross-session recovery, compaction survival, deterministic search
- Obsidian/OpenSpec: human-readable files, navigable, version-controllable

Write to Memory (per `memory-convention.md`) AND to Obsidian/OpenSpec filesystem for every artifact.

Read priority: Memory first; fall back to filesystem if Memory returns no results.
Write behavior: both writes MUST succeed for the operation to be complete.
Token cost warning: hybrid consumes MORE tokens per operation. Use only when you need both cross-session persistence AND local file artifacts.

## Memory API Reference

Base URL: `http://127.0.0.1:8192`

| Action | Method | Endpoint | Key Fields |
|--------|--------|----------|------------|
| Store | POST | `/api/memories` | `{content, title, topic_key, type, project, tags}` |
| Search | POST | `/api/memories/search` | `{query, tags?, limit?}` |
| Get by ID | GET | `/api/memories/{id}` | — |
| Delete | DELETE | `/api/memories/{id}` | — |
| Health | GET | `/api/health` | — |

## Obsidian CLI Reference

When `obsidian` CLI is available (requires Obsidian desktop running), use it for vault operations:

| Action | Command |
|--------|---------|
| Create note | `obsidian create name="sdd/{change-name}/{artifact}" folder="sdd/{change-name}" content="{markdown}"` |
| Read note | `obsidian read file="sdd/{change-name}/{artifact}"` |
| Search vault | `obsidian search query="sdd/{change-name}" format=json` |
| Append to daily note | `obsidian daily:append content="- [x] Completed task X in {change-name}"` |
| List tasks | `obsidian tasks file="sdd/{change-name}/tasks"` |
| Open note | `obsidian open file="sdd/{change-name}/{artifact}"` |

If CLI is not available, fall back to **direct filesystem** reads/writes on the vault directory (see `obsidian-convention.md`).

## OpenSpec CLI

When `openspec` CLI is installed, use it for structured operations in `openspec`/`hybrid` modes:

| Action | Command |
|--------|---------|
| Check artifact status | `openspec status --change "{change-name}" --json` |
| Get enriched instructions | `openspec instructions {artifact-id} --change "{change-name}" --json` |
| Archive change | `openspec archive "{change-name}"` |
| Validate | `openspec validate "{change-name}"` |

If `openspec` CLI is not available, fall back to direct file reads per `openspec-convention.md`.

## State Persistence (Orchestrator)

The orchestrator persists DAG state after each phase transition to enable SDD recovery after compaction.

| Mode | Persist State | Recover State |
|------|--------------|---------------|
| `memory` | `POST /api/memories` with `topic_key: "sdd/{change-name}/state"` | Search `sdd/*/state`, then GET by ID |
| `obsidian` | Write `sdd/{change-name}/state.md` with frontmatter | Read `sdd/{change-name}/state.md` |
| `openspec` | Write `openspec/changes/{change-name}/state.yaml` | Read `openspec/changes/{change-name}/state.yaml` |
| `hybrid` | Both: Memory AND write state file | Memory first; filesystem fallback |
| `none` | Not possible — warn user | Not possible |

## Common Rules

- `none` → do NOT create or modify any project files; return results inline only
- `memory` → do NOT write any project files; persist to Memory API and return memory IDs
- `obsidian` → write Markdown files to `sdd/` directory in the vault; include YAML frontmatter
- `openspec` → write files ONLY to paths defined in `openspec-convention.md`
- `hybrid` → persist to BOTH Memory AND filesystem; follow both conventions
- NEVER force directory creation unless the mode explicitly requires it
- If unsure which mode to use, default to `none`

## Sub-Agent Context Rules

Sub-agents launch with a fresh context and NO access to the orchestrator's instructions or memory protocol.

Who reads, who writes:
- Non-SDD (general task): orchestrator searches persisted backend, passes summary in prompt; sub-agent saves discoveries
- SDD (phase with dependencies): sub-agent reads artifacts directly from the active backend; sub-agent saves its artifact
- SDD (phase without dependencies, e.g. explore): nobody reads; sub-agent saves its artifact

Why this split:
- Orchestrator reads for non-SDD: it knows what context is relevant; sub-agents doing their own searches waste tokens on irrelevant results
- Sub-agents read for SDD: SDD artifacts are large; inlining them in the orchestrator prompt would consume the entire context window
- Sub-agents always write: they have the complete detail on what happened; nuance is lost by the time results flow back to the orchestrator

## Orchestrator Prompt Instructions for Sub-Agents

Non-SDD:
```
PERSISTENCE (MANDATORY):
If you make important discoveries, decisions, or fix bugs, you MUST save them before returning.

If mode is `memory`:
  curl -s -X POST http://127.0.0.1:8192/api/memories \
    -H "Content-Type: application/json" \
    -d '{"content": "{What, Why, Where, Learned}", "title": "{short-kebab-description}", "type": "{decision|bugfix|discovery|pattern}", "project": "{project}", "tags": ["{type}", "{project}"]}'

If mode is `obsidian`:
  obsidian daily:append content="- {decision/discovery summary}"
  obsidian create name="decisions/{short-kebab}" folder="decisions" content="{frontmatter + What/Why/Where/Learned}"

Do NOT return without saving what you learned. This is how the team builds persistent knowledge across sessions.
```

SDD (with dependencies) — memory mode:
```
Artifact store mode: memory
Read these artifacts before starting (search returns previews, MUST retrieve full content):

Read Step 1 — Search:
  curl -s -X POST http://127.0.0.1:8192/api/memories/search \
    -H "Content-Type: application/json" \
    -d '{"query": "sdd/{change-name}/{artifact-type}", "tags": ["sdd"], "limit": 5}'

Read Step 2 — Full retrieval (REQUIRED):
  curl -s http://127.0.0.1:8192/api/memories/{id}

PERSISTENCE (MANDATORY — do NOT skip):
After completing your work, you MUST call:
  curl -s -X POST http://127.0.0.1:8192/api/memories \
    -H "Content-Type: application/json" \
    -d '{"content": "{your full artifact markdown}", "title": "sdd/{change-name}/{artifact-type}", "topic_key": "sdd/{change-name}/{artifact-type}", "type": "architecture", "project": "{project}", "tags": ["sdd", "{artifact-type}", "{change-name}"]}'
If you return without saving, the next phase CANNOT find your artifact and the pipeline BREAKS.
```

SDD (with dependencies) — obsidian mode:
```
Artifact store mode: obsidian
Read these artifacts before starting:

If Obsidian CLI available:
  obsidian read file="sdd/{change-name}/{artifact-type}"
Else (filesystem fallback):
  cat "{vault-root}/sdd/{change-name}/{artifact-type}.md"

PERSISTENCE (MANDATORY — do NOT skip):
After completing your work, you MUST write your artifact:

If Obsidian CLI available:
  obsidian create name="sdd/{change-name}/{artifact-type}" folder="sdd/{change-name}" content="{frontmatter + markdown}"
Else (filesystem fallback):
  Write to {vault-root}/sdd/{change-name}/{artifact-type}.md with frontmatter

If you return without saving, the next phase CANNOT find your artifact and the pipeline BREAKS.
```

SDD (no dependencies):
```
Artifact store mode: {memory|obsidian|openspec|hybrid|none}

PERSISTENCE (MANDATORY — do NOT skip):
After completing your work, you MUST persist your artifact per the active mode convention.
If you return without saving, the next phase CANNOT find your artifact and the pipeline BREAKS.
```

## Skill Registry

The orchestrator pre-resolves compact rules from the skill registry and injects them as `## Project Standards (auto-resolved)` in your launch prompt. Sub-agents do NOT read the registry or individual SKILL.md files — rules arrive pre-digested.

To generate/update: run the `skill-registry` skill, or run `sdd-init`.

Sub-agent skill loading: check for a `## Project Standards (auto-resolved)` block in your prompt — if present, follow those rules. If not present, check for `SKILL: Load` instructions as a fallback. If neither exists, proceed without — this is not an error.

## Detail Level

The orchestrator may pass `detail_level`: `concise | standard | deep`. This controls output verbosity but does NOT affect what gets persisted — always persist the full artifact.