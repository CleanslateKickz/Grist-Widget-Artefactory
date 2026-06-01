# TaskFlow Design System

## 1. Widget Comparison Matrix

| Aspect | Kanban v14 | Gantt v14 | Calendar v21 | Dashboard v2 |
|--------|------------|-----------|--------------|--------------|
| **Font-family** | Inter, -apple-system, BlinkMacSystemFont | Inter, -apple-system, BlinkMacSystemFont | Inter, -apple-system, BlinkMacSystemFont, Segoe UI | -apple-system, BlinkMacSystemFont, Segoe UI, Roboto |
| **H1 font-size** | 1.25rem | 1.15rem | 1.25rem | 1.1rem |
| **Btn padding** | 8px 12px | 6px 12px | 8px 12px | 6px 12px |
| **Btn border-radius** | 8px | 6px | 8px | 6px |
| **Btn font-size** | 0.8rem | 0.75rem | 0.8rem | 0.8rem |
| **Panel width** | 340px | 340px | 340px | N/A |
| **--text-light** | ✅ | ✅ | ✅ | ❌ |
| **--border-dark** | ✅ | ✅ | ✅ | ❌ |
| **--shadow** | ✅ | ✅ | ✅ | ❌ |
| **--shadow-lg** | ✅ | ✅ | ✅ | ❌ |
| **--*-light colors** | ❌ | ❌ | ❌ | ✅ (success, warning, danger, info) |
| **SortableJS** | ✅ | ✅ | ❌ | ❌ |
| **html2canvas** | ✅ (dynamic) | ✅ (dynamic) | ✅ (dynamic) | ✅ (dynamic) |
| **Print CSS** | ✅ | ✅ | ✅ | ✅ |

---

## 2. Standard Design Tokens (To Adopt)

```css
:root {
    /* === Primary === */
    --primary: #4f46e5;
    --primary-dark: #4338ca;
    --primary-light: #e0e7ff;

    /* === Semantic Colors === */
    --success: #10b981;
    --success-light: #d1fae5;
    --warning: #f59e0b;
    --warning-light: #fef3c7;
    --danger: #ef4444;
    --danger-light: #fee2e2;
    --info: #3b82f6;
    --info-light: #dbeafe;

    /* === Backgrounds === */
    --bg: #f8fafc;
    --card-bg: #ffffff;

    /* === Text === */
    --text: #1e293b;
    --text-muted: #64748b;
    --text-light: #94a3b8;

    /* === Borders === */
    --border: #e2e8f0;
    --border-dark: #cbd5e1;

    /* === Shadows === */
    --shadow: 0 1px 3px rgba(0,0,0,0.1);
    --shadow-lg: 0 10px 40px rgba(0,0,0,0.15);

    /* === Layout === */
    --panel-width: 340px;
}
```

---

## 3. Typography Standard

```css
/* Font Stack */
body {
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

/* Headings */
h1 { font-size: 1.25rem; font-weight: 700; }

/* Text Sizes */
.text-xs   { font-size: 0.65rem; }  /* Meta, badges */
.text-sm   { font-size: 0.75rem; }  /* Labels, buttons */
.text-base { font-size: 0.85rem; }  /* Body text */
.text-lg   { font-size: 1rem; }     /* Titles */
.text-xl   { font-size: 1.25rem; }  /* Headers */
```

---

## 4. Standard Components

### 4.1 Buttons

```css
.btn {
    padding: 8px 12px;
    border-radius: 8px;
    border: 1px solid var(--border);
    font-weight: 500;
    cursor: pointer;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
    font-size: 0.8rem;
    transition: all 0.15s;
    background: var(--card-bg);
    color: var(--text);
}
.btn:hover { background: var(--bg); }
.btn.primary { background: var(--primary); color: white; border-color: var(--primary); }
.btn.primary:hover { background: var(--primary-dark); }
```

### 4.2 Form Inputs

```css
.form-input, .form-select, .form-textarea {
    width: 100%;
    padding: 10px 12px;
    border: 1px solid var(--border);
    border-radius: 8px;
    font-size: 0.85rem;
    color: var(--text);
    background: var(--card-bg);
    transition: border-color 0.15s, box-shadow 0.15s;
}
.form-input:focus, .form-select:focus, .form-textarea:focus {
    outline: none;
    border-color: var(--primary);
    box-shadow: 0 0 0 3px var(--primary-light);
}
```

### 4.3 Priority Pills

```css
.priority-pill {
    flex: 1;
    padding: 8px 4px;
    border: 2px solid var(--border);
    border-radius: 8px;
    text-align: center;
    cursor: pointer;
    font-size: 0.7rem;
    font-weight: 600;
    background: var(--card-bg);
    transition: all 0.15s;
}
.priority-pill.p1 { border-color: var(--danger); color: var(--danger); }
.priority-pill.p1.selected { background: var(--danger); color: white; }
.priority-pill.p2 { border-color: var(--warning); color: var(--warning); }
.priority-pill.p2.selected { background: var(--warning); color: white; }
.priority-pill.p3 { border-color: var(--info); color: var(--info); }
.priority-pill.p3.selected { background: var(--info); color: white; }
.priority-pill.p4 { border-color: var(--text-muted); color: var(--text-muted); }
.priority-pill.p4.selected { background: var(--text-muted); color: white; }
```

### 4.4 Cards

```css
.card {
    background: var(--card-bg);
    border-radius: 12px;
    border: 1px solid var(--border);
    padding: 16px;
    box-shadow: var(--shadow);
}
```

### 4.5 Panel (Side Drawer)

```css
.panel {
    position: absolute;
    top: 0;
    right: 0;
    width: var(--panel-width);
    height: 100%;
    background: var(--card-bg);
    border-left: 1px solid var(--border);
    display: flex;
    flex-direction: column;
    transform: translateX(100%);
    transition: transform 0.25s ease;
    z-index: 50;
}
.panel.open { transform: translateX(0); }
```

---

## 5. Priorities (Semantic Classes)

| Class | Meaning | Color | Variable |
|--------|---------|---------|----------|
| `.p1` | Critical | Red | `--danger` (#ef4444) |
| `.p2` | High | Orange/Amber | `--warning` (#f59e0b) |
| `.p3` | Medium | Blue | `--info` (#3b82f6) |
| `.p4` | Low | Gray | `--text-muted` (#64748b) |

---

## 6. Statuses

| Status | Label | Icon |
|--------|-------|------|
| `todo` | To do | ⭕ |
| `inprogress` | In progress | 🔄 |
| `review` | In review | 👁️ |
| `done` | Done | ✅ |

---

## 7. Task Types

| Type | Label | Icon |
|------|-------|------|
| `tache` | Task | 📋 |
| `jalon` | Milestone | ◆ |
| `reunion` | Meeting | 👥 |

---

## 8. Harmonization Plan

### 8.1 Dashboard (High Priority)

- [ ] Add `--primary-dark: #4338ca`
- [ ] Add `--text-light: #94a3b8`
- [ ] Add `--border-dark: #cbd5e1`
- [ ] Add `--shadow` and `--shadow-lg`
- [ ] Change font-family → `'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif`
- [ ] Align h1 to `1.25rem`
- [ ] Align `.btn` to padding `8px 12px` and border-radius `8px`

### 8.2 Gantt (Medium Priority)

- [ ] Add the `-light` colors (success, warning, danger, info)
- [ ] Align h1 to `1.25rem` (currently 1.15rem)
- [ ] Align `.btn` to padding `8px 12px` and border-radius `8px`

### 8.3 Calendar (Low Priority)

- [ ] Add the `-light` colors (success, warning, danger, info)
- [ ] Check component consistency

### 8.4 Kanban (Reference)

- [ ] Add the `-light` colors (success, warning, danger, info)
- [ ] Kanban v14 serves as the reference for other widgets

---

## 9. Responsive Breakpoints

```css
/* Desktop first approach */
@media (max-width: 900px) {
    :root { --panel-width: 320px; }
}

@media (max-width: 700px) {
    :root { --panel-width: 100%; }
    .panel { left: 0; width: 100%; }
}
```

---

## 10. Print/Export Styles

```css
@media print {
    body { background: white !important; }
    .header, .panel, .demo-badge, .filter-dropdown, .btn { display: none !important; }
    .card, .gantt-bar, .event-bar {
        print-color-adjust: exact;
        -webkit-print-color-adjust: exact;
    }
    @page { size: landscape; margin: 1cm; }
}
```

---

## 11. Accessibility

### Focus States
```css
:focus-visible {
    outline: 2px solid var(--primary);
    outline-offset: 2px;
}
```

### Contrast Ratios
- Text on white: `--text` (#1e293b) = 12.6:1 ✅
- Muted text on white: `--text-muted` (#64748b) = 4.7:1 ✅
- Primary on white: `--primary` (#4f46e5) = 5.5:1 ✅

---

## 12. External Dependencies

| Lib | CDN URL | Used by |
|-----|---------|---------|
| Grist Plugin API | `https://docs.getgrist.com/grist-plugin-api.js` | All |
| SortableJS | `https://cdn.jsdelivr.net/npm/sortablejs@1.15.0/Sortable.min.js` | Kanban, Gantt |
| html2canvas | `https://cdnjs.cloudflare.com/ajax/libs/html2canvas/1.4.1/html2canvas.min.js` | All (loaded dynamically) |

---

## 13. CSS Class Naming Convention

### Per-Widget Prefixes
- `.kanban-*` — Kanban-specific components
- `.gantt-*` — Gantt-specific components
- `.month-*`, `.week-*`, `.year-*` — Calendar-specific components
- `.kpi-*`, `.workload-*`, `.upcoming-*` — Dashboard-specific components

### Shared Classes
- `.btn`, `.btn.primary` — Buttons
- `.card` — Containers
- `.panel`, `.panel-*` — Side panel
- `.form-*` — Forms
- `.priority-*`, `.p1`-`.p4` — Priorities
- `.status-*` — Statuses
- `.filter-*` — Filters
- `.toast`, `.save-indicator` — Notifications
