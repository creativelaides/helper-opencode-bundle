# Helper OpenCode Bundle

> A curated bundle of **OpenCode skills, plugins, and configuration** for Spec-Driven Development (SDD) and AI agent team workflows.

This is a **fork** of [Gentleman Programming's Agent Teams Lite](https://github.com/Gentleman-Programming/agent-teams-lite) (now archived), packaged as a ready-to-use bundle for [OpenCode](https://opencode.ai).

> **Note**: The original repo has been archived in favor of [gentle-ai](https://github.com/Gentleman-Programming/gentle-ai). This bundle preserves and extends the SDD skills, team utilities, and memory plugin as a standalone, portable package that you can drop into any OpenCode installation.

---

## What's Inside

### SDD Pipeline — 9 Skills (v2.1)

The Spec-Driven Development workflow: an orchestrator coordinates 9 specialized sub-agents through structured phases.

```
init → explore → propose → spec → design → tasks → apply → verify → archive
```

| Skill | Purpose | License |
|-------|---------|---------|
| `sdd-init` | Detect project stack, initialize persistence backend, build skill registry | MIT |
| `sdd-explore` | Investigate codebase, analyze approaches, return structured analysis | MIT |
| `sdd-propose` | Create change proposals with intent, scope, approach, and rollback plan | MIT |
| `sdd-spec` | Write delta specs (ADDED/MODIFIED/REMOVED) with Given/When/Then + RFC 2119 | MIT |
| `sdd-design` | Technical design with architecture decisions, data flow, file changes | MIT |
| `sdd-tasks` | Break down into actionable tasks by phase with checklist format | MIT |
| `sdd-apply` | Implement code (supports TDD RED→GREEN→REFACTOR and Standard modes) | MIT |
| `sdd-verify` | Validate with real execution: tests, build, compliance matrix per scenario | MIT |
| `sdd-archive` | Sync delta→main specs, move change to dated archive folder | MIT |

### Team Utilities — 4 Skills

| Skill | Purpose | License |
|-------|---------|---------|
| `branch-pr` | PR creation workflow with issue-first enforcement | Apache-2.0 |
| `issue-creation` | GitHub issue creation workflow (bugs & features) | Apache-2.0 |
| `judgment-day` | Parallel adversarial review: 2 blind judges, re-judge until both pass or escalate | Apache-2.0 |
| `go-testing` | Go testing patterns including Bubbletea TUI and teatest | Apache-2.0 |

### Meta Skills — 2 Skills

| Skill | Purpose | License |
|-------|---------|---------|
| `skill-creator` | Create new AI agent skills following the Agent Skills spec | Apache-2.0 |
| `skill-registry` | Generate/update `.atl/skill-registry.md` for skill discovery | MIT |

### Shared Conventions (`_shared/`)

| File | Purpose |
|------|---------|
| `memory-convention.md` | Naming and persistence conventions for Memory (mcp-memory-service) |
| `obsidian-convention.md` | Frontmatter and wikilink conventions for Obsidian vault integration |
| `openspec-convention.md` | Directory structure and file conventions for openspec mode |
| `persistence-contract.md` | Multi-backend persistence contract (memory/obsidian/openspec/hybrid/none) |
| `sdd-phase-common.md` | Shared sections: skill loading, retrieval, persistence, return envelope |
| `skill-resolver.md` | Automatic skill resolution and compact rule extraction |

### Plugin: memory-plugin.js

An [OpenCode plugin](https://opencode.ai/docs/plugins/) that injects context from a Memory Service (mcp-memory) into every session's system prompt.

**Features:**
- Auto-searches memories by project queries on session creation
- Deduplicates and sorts by relevance score
- Re-injects reduced context on session compaction
- Configurable via JSON file + environment variables
- Timeout-safe with graceful fallback when Memory is unavailable

---

## Installation

### Quick Install

```bash
# Clone into your OpenCode config directory
git clone https://github.com/creativelaides/helper-opencode-bundle ~/.config/opencode
```

### Manual Install

1. Copy the `skills/` folder to `~/.config/opencode/skills/`
2. Copy the `plugins/` folder to `~/.config/opencode/plugins/`
3. Copy `opencode.json.example` → `~/.config/opencode/opencode.json`
4. Copy `plugins/memory-plugin.json.example` → `~/.config/opencode/plugins/memory-plugin.json`
5. Edit `opencode.json` — adjust model providers and agent models to your setup
6. Edit `memory-plugin.json` — adjust the Memory endpoint if needed
7. Restart OpenCode

### Configuration

The `opencode.json.example` includes a full SDD orchestrator + 9 sub-agent configuration. Key things to customize:

- **Models**: Change `model` fields to match your available providers (e.g., `ollama-cloud/glm-5.1`, `anthropic/claude-sonnet-4-20250514`, etc.)
- **Memory**: Set `mcp.memory.enabled` to `true` and adjust the command path to your `memory` binary
- **Plugin path**: Update the `plugin` URL to point to your local `memory-plugin.js`

---

## Persistence Modes

Each SDD skill supports multiple persistence backends:

| Mode | Description |
|------|-------------|
| `memory` | Stores artifacts in mcp-memory-service (port 8192). No project files created. |
| `obsidian` | Writes to Obsidian vault with frontmatter and wikilinks. Falls back to filesystem if CLI unavailable. |
| `openspec` | Creates `openspec/` directory with specs, changes, and archive. File-based source of truth. |
| `hybrid` | Persists to both Memory AND filesystem. Best of both worlds. |
| `none` | Ephemeral only — artifacts lost when session ends. |

The orchestrator auto-resolves the mode: Memory (port 8192 healthy) > Obsidian (CLI exits 0) > none.

---

## SDD Workflow at a Glance

```
┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────┐    ┌────────┐
│  INIT    │───→│ EXPLORE  │───→│ PROPOSE  │───→│ SPEC │───→│ DESIGN │
└─────────┘    └──────────┘    └──────────┘    └──────┘    └────────┘
                                                                    │
┌─────────┐    ┌──────────┐    ┌──────────┐    ┌───────┐           │
│  ARCHIVE │←──│  VERIFY  │←──│  APPLY   │←──│ TASKS │←──────────┘
└─────────┘    └──────────┘    └──────────┘    └───────┘
```

1. **Init** — Detect stack, bootstrap persistence
2. **Explore** — Investigate the codebase, compare approaches
3. **Propose** — Define what and why (budget: <400 words)
4. **Spec** — Write behavioral requirements (budget: <650 words)
5. **Design** — Document how (budget: <800 words)
6. **Tasks** — Break into phases with checkboxes (budget: <530 words)
7. **Apply** — Implement code (TDD or Standard)
8. **Verify** — Run tests, build, validate spec compliance
9. **Archive** — Merge specs, archive the change

---

## Bundled Software

This bundle includes [memory-plugin.js](https://opencode.ai/docs/plugins/) which depends on `@opencode-ai/plugin`. Install dependencies with:

```bash
cd ~/.config/opencode && npm install
```

---

## Acknowledgments

- **Gentleman Programming** — Original author of [Agent Teams Lite](https://github.com/Gentleman-Programming/agent-teams-lite), the foundation of this bundle
- **OpenCode** — The AI coding platform this bundle is designed for

---

## License

Apache License 2.0 — see [LICENSE](LICENSE).

This bundle includes skills under both Apache-2.0 and MIT licenses. Both are compatible with the Apache-2.0 license of this distribution.