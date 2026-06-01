# Project: Collaborative Whiteboard

## Context

Grist widget for an interactive whiteboard for facilitating collaborative sessions (retrospectives, workshops, brainstorming). Originally designed for CEREMA Méditerranée / GIDI.

Key features:
- Stickies (post-it notes) with colors, text, author
- Connectors between stickies
- Zones (clusters) for grouping elements
- Dot-voting (red dot voting)
- Session timer
- Properties panel
- Navigation minimap
- Collaborative mode via Grist polling (3s)
- Magnetic grid, zoom, fit-to-screen

## Architecture

```
projects/whiteboard/
├── index.html        # Complete widget (self-contained, ~1200 lines)
└── CLAUDE.md         # This file
```

The widget is **entirely standalone** — inline HTML/CSS/JS, no framework.

## Grist Schema

Auto-created table: **`Whiteboard_Elements`**

| Column   | Type   | Usage                                    |
|----------|--------|------------------------------------------|
| `Type`    | Text   | `sticky` / `connector` / `zone`          |
| `Texte`   | Text   | Sticky content or zone label            |
| `X`       | Numeric| X position on canvas (logical px)        |
| `Y`       | Numeric| Y position                               |
| `Largeur` | Numeric| Dimensions                               |
| `Hauteur` | Numeric| Dimensions                               |
| `Couleur` | Text   | Background hex color of the sticky       |
| `Cluster` | Text   | Parent cluster/zone name                  |
| `Votes`   | Numeric| Dot-voting vote count                     |
| `Auteur`  | Text   | Author identifier                        |
| `Liens`   | Text   | Free references (e.g.: task IDs)          |

## Project-Specific Conventions

- **3s polling**: anti-loop via `_writing` flag + 800ms debounce
- **SVG canvas**: elements are `<g>` SVG groups with foreignObject for editable text
- **Coordinates**: internal canvas space (transformed by viewBox pan/zoom)
- **Grist API**: 100% via `docApi.applyUserActions` (AddRecord / UpdateRecord / RemoveRecord)
- **Introspection**: via `_grist_Tables_column` to verify existing column types

## Current Status (v1.0 — 2026-02)

✅ Complete stickies CRUD (create, inline edit, move, resize, delete)
✅ Connectors between stickies (arrows)
✅ Zones / clusters with resize
✅ Dot-voting with results panel
✅ Session timer (presets 1/3/5/10/15/20m)
✅ Properties panel (color, text, cluster, links)
✅ Multi-selection with selection rectangle
✅ Minimap
✅ Magnetic grid + snap
✅ Demo mode (without Grist)
✅ TaskFlow aesthetic adaptation (indigo palette, Inter, border-radius)

## Vision: Spatial Hierarchical Whiteboard — 5th Pillar of TaskFlow

The Whiteboard is designed to become a **contextual, spatial, and navigable workspace**, complementary to the 3 TaskFlow views:

```
Kanban    → Status view (columns)
Gantt     → Time view (horizontal bars)
Calendar  → Date view (calendar grid)
Whiteboard → Spatial hierarchical view — zoom navigation
              ├── Level 0: free canvas (project view, quarterly timeline grid)
              ├── Level 1: zone/project whiteboard (monthly/weekly view)
              ├── Level N: task/element whiteboard (daily view)
              │     ├── Free stickies + notes + moodboard
              │     ├── Media elements (iframe, image, link, markdown)
              │     └── Slider: multi-whiteboards per element
              └── Unified timeline grid — consistent across levels
```

**What makes it unique**: no existing tool (Miro, FigJam, Notion, Figma, Prezi) combines structured data + spatial zoom-in navigation + context hierarchy + timeline grid.

| Criterion | Miro/FigJam | Notion | This Whiteboard |
|-----------|------------|--------|-----------------|
| Spatial canvas | ✓ | — | ✓ |
| Linked data (Grist) | — | — | ✓ |
| Hierarchical zoom-in navigation | — | — | ✓ |
| Timeline grid | — | — | ✓ |
| Project/task/sub-task hierarchy | — | ✓ | ✓ |
| Moodboard attached to elements | — | — | ✓ |

---

## Study: TaskFlow Integration

### Objective

Enable the whiteboard to communicate with TaskFlow's Kanban / Gantt / Calendar widgets via the standard Grist **cross-selection** mechanism.

### Available Grist Mechanism

```javascript
// Broadcast (all widgets)
grist.setSelectedRows([taskId]);

// Reception (all widgets)
grist.onRecord((record) => { /* record.id = selected task */ });
```

### Concept: "Sticky Linked to a Task"

The `Liens` column of a sticky can store a TaskFlow task's numeric ID.

**Example**: a sticky with `Liens = "42"` is associated with task #42 in Kanban.

### Integration Direction 1 — Whiteboard → TaskFlow (simple)

**Trigger**: click on a sticky whose `Liens` contains an integer.

```javascript
// In the sticky click handler
const taskId = parseInt(sticky.liens);
if (!isNaN(taskId)) {
    grist.setSelectedRows([taskId]);
    // → Kanban/Gantt/Calendar automatically highlight the task
}
```

**Complexity**: very low — 3 lines in the existing selection handler.

### Integration Direction 2 — TaskFlow → Whiteboard (moderate)

**Trigger**: another widget selects a task.

```javascript
grist.onRecord((record) => {
    if (!record?.id) return;
    const linked = elements.find(e => parseInt(e.liens) === record.id);
    if (linked) {
        // Center the canvas on this sticky
        centerOnElement(linked);
        // Temporary visual highlight
        highlightElement(linked.id);
    }
});
```

**Complexity**: moderate — requires a `centerOnElement()` function that calculates the pan needed to bring the element to the center of the viewport.

### Integration Direction 3 — Quick Sticky Creation from a Task (advanced)

Future option: a "Create sticky" button in TaskFlow that pre-fills a sticky with the task title and sets `Liens = task.id`.

Would require non-standard inter-widget communication (not recommended with the current architecture — Grist does not allow writing to another widget's table directly).

#---

## Intelligent Timeline Grid

### Concept

The current pixel grid (`--grid-c`/`--grid-maj`, 25px/125px) can be replaced by a timeline grid where the X axis = time.

### Zoom-Adaptive Behavior

```
Zoom < 20%   → Quarters + Years  (Q1/Q2/Q3/Q4)
Zoom 20-50%  → Months           (Jan, Feb, Mar...)
Zoom 50-150% → Weeks            (Wk 12, Wk 13...)
Zoom 150%+   → Days             (Mon 17, Tue 18...)
```

### Temporal Position Calculation

```javascript
// Convert a date to canvas X coordinate
function dateToX(date) {
    const msPerDay = 86400000;
    const origin = new Date('2024-01-01');  // configurable origin date
    const days = (date - origin) / msPerDay;
    return days * PIXELS_PER_DAY;  // PIXELS_PER_DAY = 80px (configurable)
}
```

### Timeline Grid Rendering

The grid is rendered in the existing `<pattern id="gl">`, dynamically replaced based on zoom level. `<text>` SVG elements display time markers.

### Task Integration

- Tasks with `dateDebut` → automatically positioned at X=dateToX(dateDebut)
- Horizontal drag → reschedule (updates `dateDebut` + `dateEcheance` keeping duration)
- Milestones (`type='jalon'`) → diamonds ◆ on the grid at their date
- Meetings (`type='reunion'`) → rectangles with hatched background + time
- Y axis free: by default free (vertical drag), or switch to "swimlanes" by project/person

### Toggle

Additional toolbar button "Timeline Grid" (calendar SVG icon). The timeline grid is optional and coexists with free stickies. Stickies without `TaskRef` remain floating (unconstrained on the X axis).

---

## Rich Elements (media type — moodboard)

### New Type in Whiteboard_Elements

```
Type = 'media'
Subtypes (stored in Couleur or via dedicated field):
  'iframe'   → URL embedded in foreignObject <iframe>
  'image'    → Image URL (<image> SVG or <img> in foreignObject)
  'link'     → URL card (title + description, Open Graph preview via proxy)
  'markdown' → Rich text rendered (not just raw text)
```

### SVG Structure of a Media Sticky

```
<g class="sg media-sticky" data-id="..." data-media-type="iframe">
  <rect class="sy">  (white background with colored corner based on type)
  <foreignObject>
    <div class="media-container">
      <!-- iframe for type=iframe -->
      <iframe src="${el.text}" sandbox="allow-scripts allow-same-origin"/>
      <!-- img for type=image -->
      <img src="${el.text}" style="width:100%;height:100%;object-fit:contain"/>
      <!-- link for type=link -->
      <a href="${el.text}" target="_blank">
        <div class="link-preview">...</div>
      </a>
    </div>
  </foreignObject>
  <rect class="sb">  (selection border)
</g>
```

### Attachment to a Task (linked groups)

A media element can be attached to a task sticky via the `Cluster` field (Ref to the task sticky). When the task is moved, its media follows.

```javascript
// In onUp() after drag, extend the group logic:
if (draggedEl.type !== 'zone') {
    // Also move media attached to this sticky
    for (const [mid, mel] of S.els) {
        if (mel.cluster === draggedId && mel.type === 'media') {
            mel.x += dx; mel.y += dy;
            GB.upd(mid, { X: mel.x, Y: mel.y });
            updDOM(mid);
        }
    }
}
```

### Automatic Media Layout Around a Task

When media is attached to a task, it automatically positions to the right or below the task sticky, in a line. Media floats freely otherwise (same behavior as free stickies).

### "Add Media" Tool

New tool in toolbar: `data-tool="media"` with sub-menu:
- 🖼 Image (URL)
- 🔗 Link
- 📄 Iframe
Click on canvas → prompt URL → media sticky created.

---

## Hierarchical Navigation — Infinite Zoom

### Core Concept: Zoom as Navigation

The canvas is not a magnifier — it's a **portal**. Zooming on an element doesn't enlarge it: it *enters it*. Each sticky/zone becomes an explorable space.

```
Level 0 — Project View (zoom 5-30%)
  ├── Zone "Alpha" ──zoom→ Level 1 (project whiteboard)
  │     ├── Sticky Task #42 ──zoom→ Level 2 (task whiteboard)
  │     │     ├── Free stickies, notes, moodboard
  │     │     └── Slider: [Brainstorming] [Sprint 3] [Meeting 02/12] [+]
  │     └── Sticky Task #43 ──zoom→ ...
  └── Zone "Beta" ──zoom→ Level 1 ...
```

### Data Architecture

#### Principle: Single table, single `ContextRef` field

No separate tables per level. All elements remain in `Whiteboard_Elements`, discriminated by `ContextRef` (nullable = root level).

```
Whiteboard_Elements :
  id=1, Type=zone,   Texte="Project Alpha", ContextRef=null   → root
  id=5, Type=sticky, Texte="Task #42",   ContextRef=1      → in Project Alpha
  id=9, Type=sticky, Texte="Brainstorm note", ContextRef=5   → in Task #42
```

| New Column   | Type | Usage |
|-------------|------|-------|
| `ContextRef` | Int  | Parent element ID (null = root) |

#### `WB_Contexts` Table — Multi-whiteboards per Element

Each element can have N named whiteboards, navigated via a slider.

```
WB_Contexts :
  id=1, ParentRef=5, Index=0, Label="Brainstorming", CreatedAt=...
  id=2, ParentRef=5, Index=1, Label="Sprint Planning"
  id=3, ParentRef=5, Index=2, Label="Meeting 02/12"
```

`ContextRef` in `Whiteboard_Elements` points to a `WB_Contexts.id`.

### Unified Coordinate Space

The timeline grid uses the **same reference frame** at all levels. `X = dateToX(date)` is identical everywhere — only the scale (zoom) changes.

```
Level 0 (project):   |------Q1------|------Q2------| zoom = 5%
  Zone Alpha: X = dateToX('2026-01-01') → dateToX('2026-06-30')

Zoom into Zone Alpha → Level 1:   zoom = 30%
  |-----Jan-----|-----Feb-----|-----Mar-----|
  Task #42: X = dateToX('2026-01-15'), width = duration * PX_PER_DAY

Zoom into Task #42 → Level 2:   zoom = 80%
  |--Wk12--|--Wk13--|--Wk14--|--Wk15--|
  Free stickies positioned freely (or time-anchored)
```

An element without a date remains **floating** (unconstrained on the X axis).

### Entry / Exit Mechanics

**Entering a context** (3 modalities):
- Scroll/pinch beyond 3× on an element → pulsing glow border → continue → enter
- Double-click on the element → immediate entry
- "Explore" button in the properties panel

**Exit** (2 natural modalities):
- Pinch out / scroll out at minimum zoom already reached → shake animation + go up one level
- Click on a breadcrumb node → direct return to that level

**Depth visual feedback**:
- Canvas background: slightly lighter at each level (--canvas-bg tinted)
- Parent element drop shadow visible at viewport edge ("you are *inside* it")
- Breadcrumb panel top-left: `Project Alpha > Task #42 > Brainstorming`

### Navigation State

```javascript
// Context state
S.context = {
    stack: [],   // [{ id, label, type, x, y, wbContextId }]
    wbIdx: 0,    // active whiteboard index for this element
};

// Enter an element
function enterContext(elementId, wbContextId) {
    const el = S.els.get(elementId);
    S.context.stack.push({ id: elementId, label: el.texte, wbContextId });
    if (el.taskRef) panToDateRange(el.dateDebut, el.dateEcheance);
    render();
}

// Exit one level
function exitContext() {
    if (S.context.stack.length === 0) return;
    S.context.stack.pop();
    render();
}
```

### Multi-Whiteboard Slider

Bottom bar visible only when inside a context:

```
← | [Brainstorming] [Sprint 3] [Meeting 02/12] | + New →
```

Each tab = a `WB_Contexts` row. Clicking loads elements filtered by this `wbContextId`.

---

## Implementation Recommendation

### Consolidated Phasing

#### Block A — TaskFlow Mode (linked data)

| Phase | Content | Effort |
|-------|---------|--------|
| **1** | `TaskRef: Int` + WB→TF selection (`setSelectedRows`) | ~3h |
| **2** | Structured task rendering (priority bar, status badge, assignee initials, progress) | ~5h |
| **3** | Intelligent timeline grid (toggle + adaptive zoom + date snap) | ~6h |
| **4** | Media elements (`iframe`, `image`, `link`, `markdown`) | ~5h |
| **5** | Linked media↔task groups (linked movement) | ~3h |
| **6** | Bidirectional TF↔WB selection (`Ref:Tasks` + extended onRecord) | ~2h |

#### Block B — Hierarchical Navigation (infinite zoom)

| Phase | Content | Prerequisites | Effort |
|-------|---------|---------------|--------|
| **7** | `ContextRef: Int` in SCHEMA + contextual filtering | Phase 1 | ~4h |
| **8** | Zoom-in entry/exit + transition animation | Phase 7 | ~6h |
| **9** | Breadcrumb panel + navigation stack | Phase 8 | ~3h |
| **10** | `WB_Contexts` table + multi-whiteboard slider | Phase 7 | ~4h |
| **11** | Temporal inheritance between contexts | Phases 3+8 | ~4h |

---

**Phase 1**:
1. Add `{ id: 'TaskRef', t: 'Int', l: 'Linked task' }` to the `SCHEMA`
2. Bootstrap auto-creates the column (code already in place)
3. In `GB.applyData()`: read `data.TaskRef?.[i] || 0` → `el.taskRef`
4. In click handler: if `el.taskRef > 0` → `grist.setSelectedRows([el.taskRef])`
5. In properties panel: "Linked task" field (numeric input)

**Phase 2**:
1. `S.taskCache = new Map()` — Tasks data cache
2. `GB.loadTaskflowData()` — fetchTable Tasks + Team + Projects if detected
3. `renTaskSticky(id, el)` — priority bar, status badge, assignee initials, progress
4. `buildConns()` extended — dependency connectors (blue, dashed)

**Phase 3 — Timeline Grid**:
1. Toolbar toggle "Timeline Grid" (calendar SVG icon)
2. `dateToX(date)` — date → canvas X coordinate conversion
3. Dynamic SVG rendering: zoom-adaptive time markers
4. Milestones (◆) and meetings (hatched background) automatically positioned on X

**Phase 4 — Media Elements**:
1. `type = 'media'` — new value in the schema Choice
2. `renMediaSticky(id, el)` — branch in `ren(id)` by subtype (iframe/image/link)
3. "Media" tool with sub-menu in toolbar
4. Drag = move like normal sticky

**Phase 5 — Linked Groups**:
1. Drag a task sticky → also moves its attached media (cluster = task id)
2. Auto-layout media around their task on creation

**Phase 6 — Bidirectional Selection**:
1. Change `TaskRef` from `Int` to `Ref:Tasks` in SCHEMA
2. Configure "Select By" on Tasks in Grist
3. Extend `onRecord` to center canvas on the sticky of the selected task

**Phase 7 — Hierarchical Contexts**:
1. Add `{ id: 'ContextRef', t: 'Int', l: 'Parent context' }` to the `SCHEMA`
2. Add `{ id: 'WbCtxId', t: 'Int', l: 'Whiteboard' }` to the `SCHEMA`
3. `GB.applyData()`: read `el.contextRef` and `el.wbCtxId`
4. `S.context = { stack: [], wbCtxId: null }` — navigation state
5. `render()` filters on `ctxFilter()`: elements whose `contextRef === currentContextId`
6. Create `WB_Contexts` table via SCHEMA (ParentRef, Index, Label, CreatedAt)

**Phase 8 — Zoom-in Entry/Exit**:
1. In `onWheel`: if zoom ≥ 3× on an element → `showEnterHint(elementId)` (glow border)
2. If zoom ≥ 5× on same element → `enterContext(elementId)` (or click on hint)
3. `enterContext(id, wbCtxId)`: push stack + pan to element time range + render
4. `exitContext()`: pop stack + pan back + render
5. Zoom to minimum → `exitContext()` with shake CSS animation
6. Double-click on sticky/zone → direct `enterContext`
7. Visual transition: CSS scale 0.9→1 + canvas background fade (tinted by depth)

**Phase 9 — Breadcrumb Panel**:
1. `<div id="breadcrumb">` fixed top-left, visible if `S.context.stack.length > 0`
2. Rendering: `Project Alpha > Task #42 > Brainstorming` — each node clickable
3. Click node N → `exitContextTo(n)` (pop to level N)
4. Icon per type: zone=⬜ sticky=📝 task=✓

**Phase 10 — Multi-Whiteboard Slider**:
1. `<div id="wbSlider">` fixed at bottom, visible if inside a context
2. Load `WB_Contexts` rows for current `parentRef`
3. Clickable tabs: `[Label 1] [Label 2] [+ New]`
4. Click tab → `S.context.wbCtxId = wbCtxId` + re-filter render
5. "New" → prompt label → `AddRecord` in `WB_Contexts` → switch to this context

**Phase 11 — Temporal Inheritance**:
1. `enterContext()`: if timeline grid active + element has `dateDebut` → auto pan X
2. Zoom inheritance: entering an element positions the viewport on its time window
3. Floating elements (no date): Y free, X free (timeline grid non-constraining for them)
4. On exit: restore the parent's pan/zoom (saved in the stack)

### Important Constraint

The whiteboard uses its own `Whiteboard_Elements` table — it is not attached to TaskFlow's `Tasks` table via the standard "Select By" configuration. Cross-selection still works because it goes through the row ID, **provided both widgets are in the same Grist document**.

For `grist.setSelectedRows()` from the whiteboard to be captured by TaskFlow:
- The TaskFlow widget must have its `Select By` configured on a widget linked to the same table, **OR** listen via `grist.onRecord()` directly.
- In practice, `setSelectedRows` emits globally and TaskFlow receives if its table is `Tasks` and the ID matches.

## Points of Attention

- The 3s polling may cause slight latency in collaborative mode
- SVG `foreignObject` has limitations in some browsers (Firefox) for inline editing — primarily tested on Chrome/Edge
- Stored X/Y coordinates are in canvas space (before transform), viewport↔canvas conversion must go through `screenToCanvas()`
- **Hierarchical navigation**: the 3s polling must filter on the current `WbCtxId` to avoid loading all elements from all contexts on every tick
- **Circular `ContextRef`**: must validate on the API side — prevent an element from being its own parent (simple validation before `AddRecord`)
- **Performance**: with N levels and many elements, consider pagination by context (loading only elements of the visible level)
- **X/Y coordinates** in child contexts: remain absolute in the global temporal space — be careful not to interpret them as relative to the parent
