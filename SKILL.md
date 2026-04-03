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
  - mcp__pencil__snapshot_layout
  - mcp__pencil__search_all_unique_properties
  - mcp__pencil__replace_all_matching_properties
  - mcp__pencil__get_style_guide_tags
  - mcp__pencil__get_style_guide
  - mcp__pencil__get_guidelines
  - mcp__pencil__set_variables
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
7. **`snapshot_layout()`** — verify computed layout; catch 0px nodes before wasting a screenshot call (see layout verification below)
8. **`get_screenshot`** — verify visually after each major section
9. **Remove `placeholder: true`** — only when the frame is fully done
10. **Update `[filename].map.md`** — immediately, if frames or components were added or removed

### Layout verification with snapshot_layout

Run `snapshot_layout(filePath)` after every `batch_design` call. It returns the computed x, y, width, height for every node after flexbox/grid resolution. A node that shows `width: 0` or `height: 0` is invisible — fix it before building content on top.

Use `problemsOnly: true` to only see nodes with layout issues (clipped, overflowing). Use `maxDepth` to limit how deep it descends — `maxDepth: 0` returns only top-level layout (useful for canvas-level checks), omit for full depth.

Common causes of 0px nodes:
- Missing explicit dimensions on a non-flex child
- A flex container with no children and no explicit size
- Conflicting width values between parent and child

If a scaffold is too broken to fix: delete the root frame and re-scaffold.

### Token auditing with search_all_unique_properties

Use `search_all_unique_properties` to audit consistency across the file. The `properties` parameter accepts these values: `fillColor`, `textColor`, `strokeColor`, `strokeThickness`, `cornerRadius`, `padding`, `gap`, `fontSize`, `fontFamily`, `fontWeight`.

```
// Find all colors in use — spot hardcoded hex values that should be tokens
search_all_unique_properties(filePath, parents: ["root"], properties: ["fillColor", "textColor"])

// Verify all text uses brand fonts
search_all_unique_properties(filePath, parents: ["root"], properties: ["fontFamily"])

// Audit spacing for grid compliance
search_all_unique_properties(filePath, parents: ["root"], properties: ["gap", "padding"])
```

To bulk-replace a value across the entire file:
```
// Always search first to know what you're replacing
replace_all_matching_properties(filePath, parents: ["root"], properties: {
  fillColor: [{ from: "#oldHex", to: "#newHex" }]
})
```

Each property type is a separate key in the `properties` object. To replace a color that appears as both `fillColor` and `textColor`, include both keys in a single call:
```
replace_all_matching_properties(filePath, parents: ["root"], properties: {
  fillColor: [{ from: "#old", to: "#new" }],
  textColor: [{ from: "#old", to: "#new" }]
})
```

Always verify with `get_screenshot` after bulk replacements.

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

---

## Perceptual design defaults

Science-backed lookup tables for consistent, professional-quality output. Use these whenever building or editing frames via `batch_design`. These are universal — they apply regardless of design system or brand.

### Typography

| Size range | Letter spacing | Line height | Notes |
|------------|---------------|-------------|-------|
| 56px+ (display) | −0.03em | 1.1× | Tight tracking for large headings |
| 32–48px (heading) | −0.015em | 1.2× | Slightly tighter than default |
| 14–18px (body) | normal | 1.5× minimum | Optimal line length: ~65 characters |
| 10–12px (caption) | +0.015em | 1.4× | Wider tracking aids small-text legibility |

Minimum font size: 10px. Below that, text becomes illegible on most screens.

### Color

| Rule | Value | Why |
|------|-------|-----|
| Disabled elements | 40% opacity | Not 50% — 40% reads as clearly inactive without being invisible |
| Hover state | min 8% lightness delta from resting state | Below 8% is imperceptible on most monitors |
| Body text on dark bg | `#E2E8F0` or `#F1F5F9` | Not pure white — reduces eye strain and halation |
| Text contrast (WCAG AA) | 4.5:1 minimum | 3:1 acceptable for large text (24px+) |
| Non-text contrast | 3:1 minimum | Icons, borders, form controls |

### Motion

| Scenario | Duration | Easing |
|----------|----------|--------|
| Floor (minimum perceptible) | 100ms | — |
| Standard transitions | 200–300ms | cubic-bezier(0.4, 0, 0.2, 1) |
| Enter/appear | 200ms | decelerate: cubic-bezier(0, 0, 0.2, 1) |
| Exit/disappear | 150ms | accelerate: cubic-bezier(0.4, 0, 1, 1) |
| Complex choreography | max 400ms | ease-in-out |

### Spacing and touch targets

| Element | Size |
|---------|------|
| Mobile touch target | 44 x 44px minimum |
| Desktop click target | 24 x 24px minimum |
| Primary buttons | 40–44px height |
| Compact buttons | 32px height |
| Card border radius | 8px max |
| Button border radius | 6px |
| Focus ring offset | 2px |

### Icons

| Rule | Value |
|------|-------|
| Minimum size | 16 x 16px |
| Stroke weight (standard) | 1.5px |
| Stroke weight (emphasis) | 2px |
| Circle icons | Shift down 1px for optical centering |
| Touch padding | Extend hit area to 44 x 44px |

---

## Common mistakes

Pitfalls that waste tool calls or produce broken output. Check this list before starting a design session.

| Mistake | Fix |
|---------|-----|
| Broad `batch_get` with `patterns: [{type: "frame"}]` on large files | Read the sidecar map first; pass specific `nodeIds` |
| Skipping `snapshot_layout` after `batch_design` | Always verify computed layout before screenshotting — catch 0px nodes early |
| Hardcoding hex colors instead of using design tokens | Call `get_variables()` first; use token values from the design file |
| Exceeding 25 operations in a single `batch_design` | Break into multiple calls by logical section |
| Placing new frames without checking empty space | Call `find_empty_space_on_canvas` before inserting |
| Assuming file contents without reading | Always `batch_get` specific nodes before editing them |
| Mixing multiple screens in one frame | Each screen gets its own top-level frame |
| Not updating the sidecar map after adding/removing frames | Update `[filename].map.md` immediately after `batch_design` changes |
| Retrying MCP calls when the WebSocket has dropped | Switch to offline mode; read the sidecar map or use binary parsing |
| Using `batch_get` with `readDepth` > 3 | Starts returning massive payloads; use 2 for layout, 3 for children |

---

## References

Additional reference files are included alongside this skill:

- **`references/tool-reference.md`** — Complete parameter docs for all Pencil MCP tools with usage patterns
- **`references/scaffolds.md`** — 9 layout archetype templates for common screen types
