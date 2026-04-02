# Pencil MCP Tool Reference

Parameter reference for all Pencil MCP tools. The main `SKILL.md` has the workflow patterns — this file has the parameter details.

Always read the sidecar map (`[stem].map.md`) before making MCP calls. The map gives you frame and component IDs without consuming MCP round-trips.

---

## get_editor_state

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `include_schema` | boolean | yes | Include the `.pen` file schema. Set `true` before first read/write in a session; `false` after schema is loaded. |

Returns: active `.pen` file path, selected node IDs, canvas dimensions, available documents.

**Always call first** to confirm what's open. Never assume.

---

## open_document

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePathOrTemplate` | string | yes | Absolute path to a `.pen` file, or `"new"` for blank canvas |

Returns the root node ID. If the file doesn't exist, Pencil creates it at that path.

---

## get_guidelines

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `category` | `"guide"` or `"style"` | no | Guideline category |
| `name` | string | no | Specific guideline name |
| `params` | object | no | Key-value pairs for required params |

Call with no params to list available guides and styles. Then call with `category` + `name` to load one.

---

## batch_get

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | yes | Path to the `.pen` file |
| `patterns` | array of objects | no | Search patterns (e.g. `[{type: "frame"}]`, `[{name: "Screen.*"}]`, `[{reusable: true}]`) |
| `nodeIds` | string[] | no | Specific node IDs to fetch (preferred over patterns for large files) |
| `parentId` | string | no | Limit search to this node's subtree |
| `readDepth` | number | no | How deep to descend. Default 1. Max recommended: 3. |
| `searchDepth` | number | no | How deep to search for pattern matches. Omit for unlimited. |
| `resolveInstances` | boolean | no | Expand `ref` nodes to show full component structure |
| `resolveVariables` | boolean | no | Show computed values instead of variable references |
| `includePathGeometry` | boolean | no | Show full path geometry (default: abbreviated) |

**Sidecar-first rule:** Always check the map for IDs before calling `batch_get`. Pass specific `nodeIds` — never use broad patterns on large files.

```
// CORRECT — targeted, fast
batch_get(filePath, nodeIds: ["UgVFI", "UNE3e"], readDepth: 3)

// WRONG — will overflow context on 100+ frame files
batch_get(filePath, patterns: [{type: "frame"}], readDepth: 3)
```

---

## batch_design

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | yes | Path to the `.pen` file |
| `operations` | string | yes | Operation script (see syntax below) |

**Max 25 operations per call.** Break larger builds into multiple calls by logical section.

### Operation syntax

```javascript
foo=I("parentId", { ...properties })           // Insert
bar=C("sourceId", "parentId", { ...props })    // Copy from existing node
R("nodeId1/nodeId2", { ...props })             // Replace
U(foo+"/childId", { ...props })                // Update (use variable refs)
D("nodeId")                                    // Delete
M("nodeId", "newParentId", index)              // Move
G("nodeId", "ai", "prompt text")              // Generate AI image
```

The `document` binding is predefined — use it as parent for top-level frames.

### Common property keys

```
width, height, x, y              — dimensions and position
fillColor                        — background color (hex)
textColor                        — text color (hex)
cornerRadius                     — border radius (array: [tl, tr, br, bl])
strokeColor, strokeThickness     — border color and width
display                          — "flex", "grid", "block"
flexDirection                    — "row", "column"
alignItems, justifyContent       — flex alignment
gap                              — flex/grid gap (px)
padding                          — padding (number or array)
fontFamily                       — font name string
fontSize                         — px value
fontWeight                       — numeric or string weight
lineHeight                       — multiplier or px
letterSpacing                    — em value string (e.g. "0.05em")
opacity                          — 0-1
overflow                         — "hidden", "visible", "scroll"
name                             — node display name in Pencil
text / content                   — text content (for text nodes)
```

---

## snapshot_layout

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | yes | Path to the `.pen` file |
| `parentId` | string | no | Only return layout for this subtree |
| `maxDepth` | number | no | Depth limit. `0` = top-level only. Omit for full depth. |
| `problemsOnly` | boolean | no | Only return nodes with layout issues (clipped, overflowing) |

Returns computed x, y, width, height for every node after flexbox/grid resolution.

**Run after every `batch_design` call.** A node at `width: 0` or `height: 0` is invisible — fix before screenshotting.

---

## get_screenshot

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | yes | Path to the `.pen` file |
| `nodeId` | string | yes | ID of the node to capture |

Returns a screenshot image. Always analyze the result for visual errors, misalignment, or glitches.

**Run after `snapshot_layout` confirms no 0px nodes** — don't waste a screenshot call on broken layout.

---

## get_variables

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | yes | Path to the `.pen` file |

Returns all variables and themes defined in the document. Use to load design tokens at session start.

---

## set_variables

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | yes | Path to the `.pen` file |
| `variables` | object | yes | Variable definitions following `.pen` schema |
| `replace` | boolean | no | `true` to replace all existing variables; default merges |

Variables don't persist between files or sessions. Re-run `set_variables` after every `open_document`. Confirm with `get_variables()` afterward.

---

## find_empty_space_on_canvas

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | yes | Path to the `.pen` file |
| `width` | number | yes | Width of space needed (px) |
| `height` | number | yes | Height of space needed (px) |
| `padding` | number | yes | Minimum gap from existing content (px) |
| `direction` | `"top"`, `"right"`, `"bottom"`, `"left"` | yes | Search direction from existing content |
| `nodeId` | string | no | Find space around this specific node (default: around all content) |

Returns x/y coordinates for the empty zone. Use these as the position for new frame inserts.

---

## search_all_unique_properties

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | yes | Path to the `.pen` file |
| `parents` | string[] | yes | Root node IDs to search from (use `["root"]` for entire file) |
| `properties` | string[] | yes | Properties to audit |

**Property enum:** `fillColor`, `textColor`, `strokeColor`, `strokeThickness`, `cornerRadius`, `padding`, `gap`, `fontSize`, `fontFamily`, `fontWeight`

Returns every unique value in use for the specified properties, recursively through all children.

```
// Audit all colors in use
search_all_unique_properties(filePath, parents: ["root"], properties: ["fillColor", "textColor"])

// Check font consistency
search_all_unique_properties(filePath, parents: ["root"], properties: ["fontFamily", "fontSize"])
```

**Always call before `replace_all_matching_properties`** — know what you're replacing.

---

## replace_all_matching_properties

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | yes | Path to the `.pen` file |
| `parents` | string[] | yes | Root node IDs to search (use `["root"]` for entire file) |
| `properties` | object | yes | Replacement map per property type |

Each property type is a key containing an array of `{ from, to }` pairs:

```
replace_all_matching_properties(filePath, parents: ["root"], properties: {
  fillColor: [{ from: "#2D6A4F", to: "#1B5E42" }],
  textColor: [{ from: "#2D6A4F", to: "#1B5E42" }]
})
```

Replaces recursively through all children. Verify with `get_screenshot` after.

---

## export_nodes

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | yes | Path to the `.pen` file |
| `nodeIds` | string[] | yes | Node IDs to export |
| `outputDir` | string | yes | Output directory (created if needed) |
| `format` | `"png"`, `"jpeg"`, `"webp"`, `"pdf"` | no | Default: `"png"` |
| `scale` | number | no | Scale factor (default: 2x). Not applicable to PDF. |
| `quality` | number | no | 1-100 for JPEG/WEBP. Default: 95 JPEG, 100 (lossless) WEBP. |

For PDF format, all nodes are combined into a single multi-page document.
