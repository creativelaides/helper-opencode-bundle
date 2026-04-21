# Memory Artifact Convention (reference documentation)

NOTE: Critical memory API calls (`POST /api/memories`, `POST /api/memories/search`, `GET /api/memories/{id}`) are inlined directly in each skill's SKILL.md. This document is supplementary reference — sub-agents do NOT need to read it to function.

## Memory Server

The mcp-memory-service HTTP API runs at `http://127.0.0.1:8192` (configurable via `MCP_HTTP_PORT`).

All API calls use `curl` or `httpx` against the REST endpoints below.

## Naming Rules

ALL SDD artifacts persisted to Memory MUST follow this deterministic naming:

```
title:     sdd/{change-name}/{artifact-type}
topic_key: sdd/{change-name}/{artifact-type}
type:      architecture
project:   {detected or current project name}
scope:     project
```

### Artifact Types

| Artifact Type | Produced By | Description |
|---------------|-------------|-------------|
| `explore` | sdd-explore | Exploration analysis |
| `proposal` | sdd-propose | Change proposal |
| `spec` | sdd-spec | Delta specifications (all domains concatenated) |
| `design` | sdd-design | Technical design |
| `tasks` | sdd-tasks | Task breakdown |
| `apply-progress` | sdd-apply | Implementation progress (one per batch) |
| `verify-report` | sdd-verify | Verification report |
| `archive-report` | sdd-archive | Archive closure with lineage |
| `state` | orchestrator | DAG state for recovery after compaction |

Exception: `sdd-init` uses `sdd-init/{project-name}` as both title and topic_key.

### State Artifact

```bash
curl -s -X POST http://127.0.0.1:8192/api/memories \
  -H "Content-Type: application/json" \
  -d '{
    "content": "change: {change-name}\nphase: {last-phase}\nartifact_store: memory\nartifacts:\n  proposal: true\n  specs: true\n  design: false\n  tasks: false\ntasks_progress:\n  completed: []\n  pending: []\nlast_updated: {ISO date}",
    "title": "sdd/{change-name}/state",
    "topic_key": "sdd/{change-name}/state",
    "type": "architecture",
    "project": "{project}",
    "tags": ["sdd", "state", "{change-name}"]
  }'
```

Recovery: search for `sdd/{change-name}/state` → retrieve full content.

## Recovery Protocol (2 steps)

```
Step 1: POST /api/memories/search — { "query": "sdd/{change-name}/{artifact-type}", "tags": ["sdd"], "limit": 5 }
Step 2: GET /api/memories/{id} → full content
```

When retrieving multiple artifacts, group all searches first, then all retrievals:

```
STEP A — SEARCH (get IDs only):
  POST /api/memories/search { "query": "sdd/{change-name}/proposal", "tags": ["sdd"], "limit": 5 } → save ID
  POST /api/memories/search { "query": "sdd/{change-name}/spec", "tags": ["sdd"], "limit": 5 } → save ID
  POST /api/memories/search { "query": "sdd/{change-name}/design", "tags": ["sdd"], "limit": 5 } → save ID

STEP B — RETRIEVE FULL CONTENT (mandatory):
  GET /api/memories/{proposal_id}
  GET /api/memories/{spec_id}
  GET /api/memories/{design_id}
```

Loading project context:
```
POST /api/memories/search { "query": "sdd-init/{project}", "tags": ["sdd-init"], "limit": 3 } → get ID
GET /api/memories/{id} → full project context
```

## Writing Artifacts

Standard write:
```bash
curl -s -X POST http://127.0.0.1:8192/api/memories \
  -H "Content-Type: application/json" \
  -d '{
    "content": "{full markdown content}",
    "title": "sdd/{change-name}/{artifact-type}",
    "topic_key": "sdd/{change-name}/{artifact-type}",
    "type": "architecture",
    "project": "{project}",
    "tags": ["sdd", "{artifact-type}", "{change-name}"]
  }'
```

Concrete example — saving a proposal for `add-dark-mode`:
```bash
curl -s -X POST http://127.0.0.1:8192/api/memories \
  -H "Content-Type: application/json" \
  -d '{
    "content": "## Proposal\n\nAdd dark mode toggle...",
    "title": "sdd/add-dark-mode/proposal",
    "topic_key": "sdd/add-dark-mode/proposal",
    "type": "architecture",
    "project": "my-app",
    "tags": ["sdd", "proposal", "add-dark-mode"]
  }'
```

Update existing artifact (upsert via topic_key — saving with the same `topic_key` updates, not duplicates):
```bash
curl -s -X POST http://127.0.0.1:8192/api/memories \
  -H "Content-Type: application/json" \
  -d '{
    "content": "{updated full content}",
    "title": "sdd/{change-name}/{artifact-type}",
    "topic_key": "sdd/{change-name}/{artifact-type}",
    "type": "architecture",
    "project": "{project}",
    "tags": ["sdd", "{artifact-type}", "{change-name}"]
  }'
```

### Browsing All Artifacts for a Change

```bash
curl -s -X POST http://127.0.0.1:8192/api/memories/search \
  -H "Content-Type: application/json" \
  -d '{"query": "sdd/{change-name}/", "tags": ["sdd"], "limit": 20}'
```

## API Endpoint Reference

| Action | Method | Endpoint | Body |
|--------|--------|----------|------|
| Store memory | POST | `/api/memories` | `{content, title, topic_key, type, project, tags}` |
| Search memories | POST | `/api/memories/search` | `{query, tags?, limit?}` |
| Get memory by ID | GET | `/api/memories/{id}` | — |
| Delete memory | DELETE | `/api/memories/{id}` | — |
| List memories | GET | `/api/memories?page=1&page_size=100` | — |
| Health check | GET | `/api/health` | — |

## Why This Convention

- Deterministic titles → recovery works by exact match
- `topic_key` → enables upserts without duplicates
- `sdd/` prefix + `tags` → namespaces all SDD artifacts
- Two-step recovery → search returns previews; full retrieval for complete content
- Lineage → archive-report includes all memory IDs for complete traceability