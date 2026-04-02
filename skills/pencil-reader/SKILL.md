---
name: pencil-reader
description: Full workflow skill for Pencil.dev .pen design files — use this whenever working with a .pen file or the Pencil MCP, whether designing new screens, updating existing artboards, reading frame structure, searching for components, or recovering when the MCP connection is unstable. Covers both online mode (Pencil MCP connected: design, edit, screenshot) and offline mode (MCP down: search frames and components from binary). Always use this skill before making any Pencil MCP call — it prevents context overflow on large design files by enforcing a sidecar index pattern.
allowed-tools:
  - mcp__pencil__get_editor_state
  - mcp__pencil__batch_get
  - mcp__pencil__batch_design
  - mcp__pencil__get_variables
  - mcp__pencil__find_empty_space_on_canvas
  - mcp__pencil__get_screenshot
  - mcp__pencil__export_nodes
  - mcp__pencil__open_document
  - Bash
  - Read
  - Write
  - Glob
---

# Pencil Reader

Complete workflow for `.pen` design files. The core idea: every `.pen` file gets a plain-markdown sidecar index (`[stem].map.md`) placed next to it that you read at the start of every session instead of querying the design file directly. This keeps large files (100+ frames, 75+ components) from overflowing context.

> **Sidecar naming rule:** strip the `.pen` extension to get the stem. `nso-to.pen` → `nso-to.map.md`. `royalti-client.pen` → `royalti-client.map.md`. Never include `.pen` in the map filename.

> **New to Pencil MCP?** Pencil is a design tool whose files (`.pen`) can be read and edited programmatically via a local MCP server. The MCP exposes tools like `get_editor_state`, `batch_get`, and `batch_design`. If the MCP isn't connected, use offline mode below to still read frame and component names from the binary file.

> ⚠️ **Cross-file refs don't work via MCP.** `batch_design` cannot set document-level imports, and `ref` nodes cannot resolve IDs across files. Keep all frames and components in a single `.pen` file per project.

---

## File layout

```
pencil/
├── my-project.pen        ← design file (binary, edit via MCP only)
└── my-project.map.md     ← sidecar index (plain markdown, always readable)
```

---

## Step 0 — Orient yourself

Find the `.pen` file(s) in the project:
```bash
find . -name "*.pen" -not -path "*/node_modules/*" 2>/dev/null
```

Check if its sidecar map exists (replace `pencil/my-project` with the actual path):
```bash
ls $(dirname path/to/file.pen)/*.map.md 2>/dev/null || echo "No map — run SETUP first"
```

- **Map exists** → read it, then run the **drift check** below before doing anything else
- **No map** → run **SETUP** to generate one before doing anything else

---

## Drift check — run this at the start of every session with an existing map

The map can go stale between sessions (frames added, renamed, or deleted outside Claude). A 30-second check prevents acting on wrong IDs.

**Step 1 — Compare counts (MCP online)**

Call `get_editor_state({ include_schema: false })` and compare its top-level node count and reusable component count against the map header:

```
Map says:   Total frames: 155 / Total reusable components: 77
MCP returns: Top-Level Nodes (158) / Reusable Components (77)
```

- **Counts match** → map is fresh, proceed
- **Counts differ** → run Step 2 before touching anything

**Step 2 — Identify the delta**

```
get_editor_state returns 158 frames, map has 155 → 3 frames added since last update
```

1. Diff the `get_editor_state` node list against the map's frame table — find the new/missing IDs
2. Update the map: add missing rows, remove stale rows, correct the header counts
3. Then proceed with the original task

**MCP offline?** Compare the Python frame count against the map header instead:

```bash
python3 - <<'EOF' path/to/file.pen
import re, sys
with open(sys.argv[1], 'rb') as f:
    data = f.read().decode('utf-8', errors='ignore')
frames = re.findall(r'"type":\s*"frame"[^}]*?"name":\s*"([^"]+)"', data)
top_level = [f for f in frames if not f.startswith('Component/')]
print(f"~{len(set(top_level))} unique frame names found")
EOF
```

If the count differs significantly from the map header, flag it and repair the map on next MCP session.

---

## SETUP — Generate the sidecar map

Do this once per project, or to rebuild after major changes.

### Option A — With MCP connected (preferred, most accurate)

1. Call `get_editor_state` — this returns every top-level frame and all reusable components with their IDs and names
2. Use that output to populate the map template below — frames go in the Frames section, `reusable: true` nodes go in Reusable Components
3. Write the file as `[stem].map.md` next to the `.pen` file — where stem is the `.pen` filename without the extension (e.g., `nso-to.pen` → `nso-to.map.md`)
4. Update the frame and component counts at the top

The map doesn't need to capture every internal child node — just top-level frames and reusable components. Those are the IDs you'll pass to `batch_get` to navigate the file efficiently.

### Option B — Without MCP (offline bootstrap)

Extract names and IDs from the binary file using Python string parsing. Results are usually accurate but may miss some IDs in unusual encodings — verify with MCP when possible.

```bash
# List all numbered artboard frames
# The regex targets the common naming pattern "1.2 — Screen Name" (with any dash/em-dash variant).
# If a project uses a different naming convention (no leading number, different separator),
# broaden the regex or dump all top-level frame names instead.
python3 - <<'EOF' path/to/file.pen
import re, sys
with open(sys.argv[1], 'rb') as f:
    data = f.read().decode('utf-8', errors='ignore')
matches = re.findall(r'"name":\s*"(\d+[\.\d]*[a-z]?\s*(?:—|─|–|-)\s*[^"]+)"', data)
seen = set()
for m in matches:
    clean = m.replace('\n', ' ').replace('  ', ' ').strip()
    if clean not in seen:
        seen.add(clean)
        print(clean)
EOF

# Extract reusable component names
python3 - <<'EOF' path/to/file.pen
import re, sys
with open(sys.argv[1], 'rb') as f:
    data = f.read().decode('utf-8', errors='ignore')
matches = re.findall(r'"name":\s*"(Component/[^"]+)"', data)
seen = set()
for m in sorted(matches):
    if m not in seen:
        seen.add(m)
        print(m)
EOF

# Find the node ID for a specific frame name
python3 - <<'EOF' path/to/file.pen "Exact Frame Name"
import re, sys
target = sys.argv[2]
with open(sys.argv[1], 'rb') as f:
    data = f.read().decode('utf-8', errors='ignore')
for pattern in [
    r'"id":\s*"([A-Za-z0-9]+)"[^}]*?"name":\s*"' + re.escape(target),
    r'"name":\s*"' + re.escape(target) + r'"[^}]*?"id":\s*"([A-Za-z0-9]+)"',
]:
    matches = re.findall(pattern, data)
    if matches:
        for m in matches:
            print(f'ID: {m}')
        break
EOF
```

---

## Map format

```markdown
# [stem].pen — Node Map

Sidecar index for `[stem].pen`. Keep in sync with the Pencil file.
**Update this file after any `batch_design` that adds or removes frames or components.**

- filePath (for MCP): `pencil/[stem].pen`
- Total frames: N
- Total reusable components: N

---

## Frames

### [Section Name]
| ID | Name |
|----|------|
| abc123 | 1.1 — Screen Name |
| def456 | 1.2 — Another Screen |

---

## Reusable Components

### Buttons
| ID | Name |
|----|------|
| xyz789 | Component/Button/Primary |
| uvw012 | Component/Button/Secondary |

### [Other categories as needed]
```

---

## ONLINE MODE — MCP Workflow

Use when `get_editor_state` returns successfully.

### Querying rules — read these before any MCP call

Large `.pen` files have 100+ frames and 75+ components. A single broad query can return 150K+ characters and hang the session. Targeted reads are both faster and reliable.

| Rule | Why it matters |
|------|----------------|
| **Read the map first, always** | Gets you all IDs in one fast `Read` call — no MCP needed |
| **Pass specific `nodeIds` to `batch_get`** | Broad pattern searches on the full document overflow context |
| **`readDepth` max 3** | Use 2 for layout checks, 3 when you need child structure |
| **`searchDepth: 1`** | If patterns are unavoidable, scope them tightly |
| **2–3 IDs per `batch_get` call** | Small batches stay fast and predictable |
| **`filePath` required on every call** | The MCP won't assume a default |

```
// CORRECT — targeted, fast
batch_get(filePath, nodeIds: ["UgVFI", "UNE3e"], readDepth: 3)

// WRONG — will overflow context and may hang
batch_get(filePath, patterns: [{reusable: true}], readDepth: 3)
batch_get(filePath, patterns: [{type: "frame"}], readDepth: 1)
```

### Design workflow

1. **`Read [filename].map.md`** — get frame and component IDs for the session
2. **Drift check** — compare map header counts against `get_editor_state` totals; repair map if they differ before proceeding
3. **`get_variables()`** — load design tokens; never hardcode colors or spacing values
4. **`batch_get(nodeIds: [2–3 IDs], readDepth: 2–3)`** — inspect existing structure before editing
5. **`find_empty_space_on_canvas`** — get coordinates for new frames; never place over existing content
6. **`batch_design`** — build in chunks of max 25 ops; mark in-progress frames with `placeholder: true`
7. **`get_screenshot`** — verify visually after each major section
8. **Remove `placeholder: true`** — only when the frame is fully done
9. **Update `[filename].map.md`** — immediately, if frames or components were added or removed

### Drift detection and repair

When `batch_get` returns `"No node with id X"` for an ID from the map, the map is out of sync.

1. Call `get_editor_state` to get the current top-level node list
2. Compare against the map — find what was added or removed
3. Update `[filename].map.md` and correct the total counts
4. Resume the original task with the corrected IDs

### Map update protocol

After `batch_design` adds or removes a top-level frame or reusable component:

1. Note the new node's ID from the operation binding (e.g. `newFrame=I(...)` → ID is in `newFrame`)
2. Add or remove the row in the correct section of `[filename].map.md`
3. Update the total counts at the top of the file

---

## OFFLINE MODE — Binary fallback

Use when the Pencil desktop app is closed, the MCP WebSocket keeps dropping, or you just need a quick lookup without starting a design session.

### Mode selection

| Task | Use |
|------|-----|
| Look up frame names | **Offline** — instant |
| Search frames by keyword | **Offline** — reliable |
| Check if a component exists | **Offline** — grep component names |
| Take a screenshot of a frame | **Online only** |
| Edit or insert nodes | **Online only** |
| Export to PNG/PDF | **Online only** |

### Search frames by keyword

```bash
python3 - <<'EOF' path/to/file.pen "upload"
import re, sys
keyword = sys.argv[2].lower()
with open(sys.argv[1], 'rb') as f:
    data = f.read().decode('utf-8', errors='ignore')
matches = re.findall(r'"name":\s*"(\d+[\.\d]*[a-z]?\s*(?:—|─|–|-)\s*[^"]+)"', data)
seen = set()
for m in matches:
    clean = m.replace('\n', ' ').replace('  ', ' ').strip()
    if keyword in clean.lower() and clean not in seen:
        seen.add(clean)
        print(clean)
EOF
```

### Search all text content

```bash
strings path/to/file.pen | grep -i "keyword" | head -30
```

### Count frames by section

```bash
python3 - <<'EOF' path/to/file.pen
import re, sys, collections
with open(sys.argv[1], 'rb') as f:
    data = f.read().decode('utf-8', errors='ignore')
matches = re.findall(r'"name":\s*"(\d+)\.\d+[a-z]?\s*(?:—|─|–|-)\s*([^"]+)"', data)
sections = collections.OrderedDict()
for num, name in matches:
    clean = name.replace('\n', ' ').strip()
    key = f'Section {num}'
    if key not in sections:
        sections[key] = []
    if clean not in sections[key]:
        sections[key].append(clean)
for section, frames in sections.items():
    print(f'{section}: {len(frames)} frames')
    for f in frames[:3]:
        print(f'  - {f}')
    if len(frames) > 3:
        print(f'  ... +{len(frames)-3} more')
EOF
```
