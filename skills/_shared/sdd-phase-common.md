# SDD Phase — Common Protocol

Boilerplate identical across all SDD phase skills. Sub-agents MUST load this alongside their phase-specific SKILL.md.

Executor boundary: every SDD phase agent is an EXECUTOR, not an orchestrator. Do the phase work yourself. Do NOT launch sub-agents, do NOT call `delegate`/`task`, and do NOT bounce work back unless the phase skill explicitly says to stop and report a blocker.

## A. Skill Loading

1. Check if the orchestrator injected a `## Project Standards (auto-resolved)` block in your launch prompt. If yes, follow those rules — they are pre-digested compact rules from the skill registry. **Do NOT read any SKILL.md files.**
2. If no Project Standards block was provided, check for `SKILL: Load` instructions. If present, load those exact skill files.
3. If neither was provided, search for the skill registry as a fallback:
   a. Search Memory API for skill-registry: `POST /api/memories/search` with `{"query": "skill-registry", "tags": ["config"], "limit": 3}` → if found, `GET /api/memories/{id}` for full content
   b. Fallback: read `.atl/skill-registry.md` from the project root if it exists
   c. From the registry's **Compact Rules** section, apply rules whose triggers match your current task.
4. If no registry exists, proceed with your phase skill only.

NOTE: the preferred path is (1) — compact rules pre-injected by the orchestrator. Paths (2) and (3) are fallbacks for backwards compatibility. Searching the registry is SKILL LOADING, not delegation. If `## Project Standards` is present, IGNORE any `SKILL: Load` instructions — they are redundant.

## B. Artifact Retrieval

### Memory Mode

**CRITICAL**: Memory search returns truncated PREVIEWS, not full content. You MUST retrieve full content via `GET /api/memories/{id}` for EVERY artifact. **Skipping this produces wrong output.**

**Run all searches in parallel** — do NOT search sequentially.

```bash
curl -s -X POST http://127.0.0.1:8192/api/memories/search \
  -H "Content-Type: application/json" \
  -d '{"query": "sdd/{change-name}/{artifact-type}", "tags": ["sdd"], "limit": 5}'
```

Then **run all retrievals in parallel**:

```bash
curl -s http://127.0.0.1:8192/api/memories/{id} → full content (REQUIRED)
```

Do NOT use search previews as source material.

### Obsidian Mode

If Obsidian CLI is available:
```bash
obsidian read file="sdd/{change-name}/{artifact-type}"
```

Filesystem fallback (always available):
```bash
# Read the note from vault — strip frontmatter for content
cat "{vault-root}/sdd/{change-name}/{artifact-type}.md"
```

### OpenSpec Mode

Read files directly per `openspec-convention.md`, or use `openspec` CLI when available.

### None Mode

Use whatever context the orchestrator passed in the prompt.

## C. Artifact Persistence

Every phase that produces an artifact MUST persist it. Skipping this BREAKS the pipeline — downstream phases will not find your output.

### Memory mode

```bash
curl -s -X POST http://127.0.0.1:8192/api/memories \
  -H "Content-Type: application/json" \
  -d '{
    "content": "{your full artifact markdown}",
    "title": "sdd/{change-name}/{artifact-type}",
    "topic_key": "sdd/{change-name}/{artifact-type}",
    "type": "architecture",
    "project": "{project}",
    "tags": ["sdd", "{artifact-type}", "{change-name}"]
  }'
```

`topic_key` enables upserts — saving again updates, not duplicates.

### OpenSpec mode

File was already written during the phase's main step. No additional action needed.

### Hybrid mode

Do BOTH: write the file to the filesystem AND POST to Memory API as above.

### None mode

Return result inline only. Do not write any files or call the Memory API.

## D. Return Envelope

Every phase MUST return a structured envelope to the orchestrator:

- `status`: `success`, `partial`, or `blocked`
- `executive_summary`: 1-3 sentence summary of what was done
- `detailed_report`: (optional) full phase output, or omit if already inline
- `artifacts`: list of artifact keys/paths/IDs written
- `next_recommended`: the next SDD phase to run, or "none"
- `risks`: risks discovered, or "None"
- `skill_resolution`: how skills were loaded — `injected` (received Project Standards from orchestrator), `fallback-registry` (self-loaded from registry), `fallback-path` (loaded via SKILL: Load path), or `none` (no skills loaded)

Example:

```markdown
**Status**: success
**Summary**: Proposal created for `{change-name}`. Defined scope, approach, and rollback plan.
**Artifacts**: Memory `sdd/{change-name}/proposal` (ID: {memory-id}) | `openspec/changes/{change-name}/proposal.md`
**Next**: sdd-spec or sdd-design
**Risks**: None
**Skill Resolution**: injected — 3 skills (react-19, typescript, tailwind-4)
(other values: `fallback-registry`, `fallback-path`, or `none — no registry found`)
```