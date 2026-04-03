# claude-skill-pencil-reader

A Claude Code skill (and plugin) for working with [Pencil.dev](https://pencil.dev) `.pen` design files via the Pencil MCP server.

## The problem

Large `.pen` files — 100+ frames, 75+ components — overflow Claude's context when queried broadly. A single `batch_get` with `patterns: [{type: "frame"}]` can return 150K+ characters and hang the session.

## The solution

A **sidecar index pattern**: every `.pen` file gets a plain-markdown `[stem].map.md` file placed next to it. Claude reads the map at session start (one fast `Read` call) instead of querying the design file. This gives all frame and component IDs without touching the MCP.

```
pencil/
├── my-project.pen        <- design file (binary, edit via MCP only)
└── my-project.map.md     <- sidecar index (plain markdown, always readable)
```

## Installation

### Option 1 — Claude Code plugin (recommended)

```bash
# Load directly from a local clone (session-only)
claude --plugin-dir /path/to/claude-skill-pencil-reader

# Or install from a marketplace that includes this plugin
claude plugin marketplace add nedjamez/claude-skill-pencil-reader
claude plugin install pencil-reader
```

### Option 2 — Vercel skills CLI

```bash
npx skills add nedjamez/claude-skill-pencil-reader
```

This symlinks the skill into `.claude/skills/pencil-reader/` automatically.

### Option 3 — Manual copy

```bash
mkdir -p .claude/skills/pencil-reader
cp skills/pencil-reader/SKILL.md .claude/skills/pencil-reader/SKILL.md
```

### Connect the Pencil MCP

Follow the [official Pencil MCP setup guide](https://docs.pencil.dev/mcp). The MCP server runs locally and connects to Claude Code via WebSocket.

### Create your first sidecar map

With the MCP connected, ask Claude:

> "We have a Pencil design file at `pencil/my-project.pen`. Can you set up the sidecar map for it?"

Claude will call `get_editor_state`, populate the map template from the skill, and write `pencil/my-project.map.md`.

## Project structure

```
claude-skill-pencil-reader/
├── .claude-plugin/
│   └── plugin.json           <- Plugin manifest (name, version, author)
├── skills/
│   └── pencil-reader/
│       └── SKILL.md          <- The skill definition
├── references/
│   ├── tool-reference.md     <- Full Pencil MCP parameter docs
│   └── scaffolds.md          <- 9 layout archetype templates
├── evals/
│   ├── evals.json            <- 8 evaluation scenarios
│   └── benchmark.json        <- Iteration 1 benchmark results
├── package.json              <- Eval runner scripts
├── README.md
└── LICENSE
```

## What the skill covers

- **Session-start drift check** — catches frames added/renamed outside Claude
- **Online mode** — MCP workflow with querying rules that prevent context overflow
- **Offline mode** — Python scripts to search frames and extract IDs from the binary when the MCP is unavailable
- **Sidecar map format** — standardized markdown template with update protocol
- **Layout verification** — `snapshot_layout` after every build, token auditing with `search_all_unique_properties`
- **Perceptual design defaults** — science-backed rules for typography, color, motion, spacing, icons
- **Common mistakes** — 10 pitfalls that waste tool calls or produce broken output
- **Tool scoping** — `allowed-tools` frontmatter limits which tools the skill can use
- **Reference docs** — complete MCP tool parameter reference + 9 scaffold templates

## Benchmark results (iteration 1)

| Config | Pass rate | Avg duration | Avg tokens |
|--------|-----------|--------------|------------|
| With skill | 93.75% | 53,849ms | 27,261 |
| Without skill | 93.75% | 91,064ms | 32,004 |
| **Delta** | **0%** | **-41%** | **-15%** |

Same accuracy, meaningfully faster and cheaper. The biggest win is eval 3 (no existing map, offline mode): without the skill it took 216s; with the skill, 71s.

## Evals

8 scenarios testing the full skill surface:

| # | Scenario | Tests |
|---|----------|-------|
| 0 | Offline frame search | Binary parsing, no MCP connection |
| 1 | Map setup (existing) | Detect existing map, recommend verify |
| 2 | Pre-design ID check | Read map, not broad MCP query |
| 3 | Map setup (offline) | Offline bootstrap, correct naming |
| 4 | Drift check detection | Compare map vs MCP counts |
| 5 | Design + map update | Add frame, update map immediately |
| 6 | MCP error recovery | Fallback to offline on connection failure |
| 7 | Wrong map name detection | Flag `file.pen.map.md` naming violation |

View benchmark results:

```bash
npm run eval:report
```

To re-run evals, install the skill locally and use the Claude Code eval framework with `evals/evals.json`.

## License

MIT
