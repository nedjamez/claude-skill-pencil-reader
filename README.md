# claude-skill-pencil-reader

A Claude Code skill for working with [Pencil.dev](https://pencil.dev) `.pen` design files via the Pencil MCP server.

## The problem

Large `.pen` files — 100+ frames, 75+ components — overflow Claude's context when queried broadly. A single `batch_get` with `patterns: [{type: "frame"}]` can return 150K+ characters and hang the session.

## The solution

A **sidecar index pattern**: every `.pen` file gets a plain-markdown `[stem].map.md` file placed next to it. Claude reads the map at session start (one fast `Read` call) instead of querying the design file. This gives all frame and component IDs without touching the MCP.

```
pencil/
├── my-project.pen        ← design file (binary, edit via MCP only)
└── my-project.map.md     ← sidecar index (plain markdown, always readable)
```

## What's included

- **`SKILL.md`** — the Claude Code skill. Drop this into your `.claude/skills/pencil-reader/` directory. It covers:
  - Session-start drift check (catches frames added/renamed outside Claude)
  - Online mode: MCP workflow with querying rules that prevent context overflow
  - Offline mode: Python scripts to search frames and extract IDs from the binary when the MCP is unavailable
  - Sidecar map format and update protocol
- **`evals/evals.json`** — 4 evaluation scenarios for testing the skill
- **`evals/benchmark.json`** — iteration 1 benchmark results (with vs. without skill)

## Setup

### 1. Install the skill

Copy `SKILL.md` into your project's Claude config:

```bash
mkdir -p .claude/skills/pencil-reader
cp SKILL.md .claude/skills/pencil-reader/SKILL.md
```

### 2. Connect the Pencil MCP

Follow the [official Pencil MCP setup guide](https://docs.pencil.dev/mcp). The MCP server runs locally and connects to Claude Code via WebSocket.

### 3. Create your first sidecar map

With the MCP connected, ask Claude:

> "We have a Pencil design file at `pencil/my-project.pen`. Can you set up the sidecar map for it?"

Claude will call `get_editor_state`, populate the map template from SKILL.md, and write `pencil/my-project.map.md`.

## Benchmark results (iteration 1)

| Config | Pass rate | Avg duration | Avg tokens |
|--------|-----------|--------------|------------|
| With skill | 93.75% | 53,849ms | 27,261 |
| Without skill | 93.75% | 91,064ms | 32,004 |
| **Delta** | **0%** | **−41%** | **−15%** |

Same accuracy, meaningfully faster and cheaper. The biggest win is eval 3 (no existing map, offline mode): without the skill it took 216s; with the skill, 71s.

## Running evals

Evals are designed for the [Claude Code agent eval framework](https://docs.anthropic.com/en/docs/claude-code/evals). Each eval in `evals/evals.json` has a prompt and 4 pass/fail assertions.

## License

MIT
