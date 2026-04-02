# Layout Scaffold Templates

9 ready-to-use layout archetypes for common screen types. Each scaffold is a `batch_design` operation list that creates the structural skeleton — add content nodes after verifying layout with `snapshot_layout`.

**Before using any scaffold:**
1. Read the sidecar map to check existing frames
2. Call `get_variables()` to load design tokens — use token values, not hardcoded hex
3. Call `find_empty_space_on_canvas` to get placement coordinates
4. After the scaffold, run `snapshot_layout()` to catch 0px nodes before adding content

**Conventions:**
- `W` = canvas width, `H` = canvas height (from `get_editor_state`)
- `SW` = sidebar width (typically 240–256px; 0 if no sidebar)
- `PW` = page area width (`W - SW`)
- All tokens (colors, fonts, spacing) come from `get_variables()` — never hardcode
- Max 25 operations per `batch_design` call — split if needed

---

## Scaffold A — Dashboard

Sidebar + page area with header, stats bar, and scrollable content zone.

**Structure:**
```
root (W x H)
├── sidebar (SW x H)
├── pageArea (PW x H, flexDirection: column)
│   ├── pageHeader (PW x 64, flexDirection: row, alignItems: center)
│   ├── statsBar (PW x 80, flexDirection: row, gap: 16)
│   │   ├── statCard x 3-4 (flex: 1, padding: 16)
│   └── contentWrap (PW x remaining, overflow: scroll)
│       └── [your content here]
```

**When to use:** Home screens, overview pages, analytics dashboards, admin panels.

---

## Scaffold B — List / Queue

Sidebar + page area with header, command strip (search + filters), and table/list body.

**Structure:**
```
root (W x H)
├── sidebar (SW x H)
├── pageArea (PW x H, flexDirection: column)
│   ├── pageHeader (PW x 64)
│   ├── commandStrip (PW x 52, flexDirection: row, alignItems: center, gap: 8)
│   │   ├── searchInput (240 x 36)
│   │   ├── filterGroup (flexDirection: row, gap: 8)
│   │   └── actionButton (aligned right)
│   └── tableBody (PW x remaining, overflow: scroll)
│       ├── tableHeader (PW x 40, flexDirection: row)
│       └── tableRows (PW x remaining)
```

**When to use:** Data tables, queues, libraries, file managers, inbox views.

---

## Scaffold C — Detail / Review

Two-panel layout (typically 40/60 split) for side-by-side viewing.

**Structure:**
```
root (W x H)
├── sidebar (SW x H)
├── pageArea (PW x H, flexDirection: column)
│   ├── pageHeader (PW x 64)
│   ├── actionBar (PW x 56, flexDirection: row, justifyContent: space-between)
│   └── splitView (PW x remaining, flexDirection: row)
│       ├── leftPanel (40% of PW, overflow: scroll, padding: 24)
│       └── rightPanel (60% of PW, overflow: scroll, padding: 24)
```

**When to use:** Profile views, review screens, email readers, document editors, comparison views.

---

## Scaffold D — Marketing Page

Full-width at 1440px regardless of app resolution. Verify at 390px mobile width.

**Structure:**
```
root (1440 x auto)
├── navbar (1440 x 72, flexDirection: row, alignItems: center, padding: 0 80)
├── hero (1440 x 600, display: flex, alignItems: center, justifyContent: center)
│   └── heroContent (768 x auto, text-align: center)
├── section1 (1440 x auto, padding: 80 0)
│   └── sectionContent (1088 x auto, margin: 0 auto)
├── section2 (1440 x auto, padding: 80 0)
│   └── sectionContent (1088 x auto, margin: 0 auto)
└── footer (1440 x auto, padding: 48 80)
```

**When to use:** Landing pages, product pages, pricing pages, about pages.

**Note:** Content max width is typically 768px for copy-heavy sections, 1088px for feature grids.

---

## Scaffold E — Modal / Dialog

Backdrop overlay with centered modal. Size varies by use case.

**Structure:**
```
backdrop (W x H, fillColor: rgba(0,0,0,0.5))
└── modal (380-720px x auto, centered, fillColor: surface, cornerRadius: [12,12,12,12], padding: 24)
    ├── modalHeader (flexDirection: row, justifyContent: space-between)
    │   ├── title (text, fontSize: 18, fontWeight: 600)
    │   └── closeButton (24 x 24)
    ├── modalBody (padding: 16 0)
    │   └── [your content here]
    └── modalFooter (flexDirection: row, justifyContent: flex-end, gap: 8)
        ├── cancelButton
        └── confirmButton
```

**Width guidelines:** 380px for confirmations, 480px for forms, 640px for complex content, 720px for data-heavy modals.

---

## Scaffold F — Wizard / Stepper

Multi-step form with header, stepper progress bar, centered form area, and pinned footer.

**Structure:**
```
root (W x H)
├── header (W x 64)
├── stepperBar (W x 60, flexDirection: row, alignItems: center, justifyContent: center, gap: 24)
│   ├── step1 (indicator + label)
│   ├── connector (line)
│   ├── step2 (indicator + label)
│   ├── connector (line)
│   └── step3 (indicator + label)
├── formArea (640 x remaining, margin: 0 auto, padding: 32, overflow: scroll)
│   └── [step content here]
└── footer (W x 72, flexDirection: row, justifyContent: space-between, padding: 0 24)
    ├── backButton
    └── nextButton
```

**When to use:** Onboarding flows, setup wizards, multi-step forms, checkout flows.

---

## Scaffold G — Mobile Screen

Fixed at 390 x 844px (iPhone 14 / standard mobile viewport).

**Structure:**
```
root (390 x 844)
├── statusBar (390 x 44, padding: 0 16)
├── navBar (390 x 56, flexDirection: row, alignItems: center, padding: 0 16)
│   ├── backButton (24 x 24)
│   ├── title (flex: 1, textAlign: center)
│   └── actionButton (24 x 24)
├── scrollContent (390 x 688, overflow: scroll, padding: 16)
│   └── [your content here]
└── bottomNav (390 x 56, flexDirection: row, justifyContent: space-around, alignItems: center)
    ├── navItem x 4-5
```

**When to use:** Any mobile app screen. Always 390 x 844 — don't vary the viewport.

**Touch targets:** All interactive elements must be at least 44 x 44px.

---

## Scaffold H — Form / Data Entry

Single-column centered form within the standard sidebar layout.

**Structure:**
```
root (W x H)
├── sidebar (SW x H)
├── pageArea (PW x H, flexDirection: column)
│   ├── pageHeader (PW x 64)
│   └── formContainer (PW x remaining, display: flex, justifyContent: center, padding: 32)
│       └── form (600 x auto, flexDirection: column, gap: 20)
│           ├── fieldGroup (flexDirection: column, gap: 4)
│           │   ├── label (fontSize: 14, fontWeight: 500)
│           │   └── input (600 x 40, cornerRadius: [6,6,6,6], strokeColor: border)
│           ├── fieldGroup x N
│           └── submitButton (alignSelf: flex-end)
```

**When to use:** Settings forms, profile editors, create/edit screens, data entry.

---

## Scaffold I — Empty State

Centered placeholder with illustration area, text stack, and call-to-action.

**Structure:**
```
root (W x H)
├── sidebar (SW x H)
├── pageArea (PW x H, flexDirection: column)
│   ├── pageHeader (PW x 64)
│   └── emptyContainer (PW x remaining, display: flex, alignItems: center, justifyContent: center)
│       └── emptyContent (400 x auto, flexDirection: column, alignItems: center, gap: 16)
│           ├── illustration (200 x 200, fillColor: background)
│           ├── heading (fontSize: 20, fontWeight: 600, textAlign: center)
│           ├── description (fontSize: 14, textColor: muted, textAlign: center, maxWidth: 320)
│           └── ctaButton (padding: 10 20, cornerRadius: [6,6,6,6])
```

**When to use:** Zero-data states, first-run experiences, search with no results, error recovery.
