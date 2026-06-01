# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**TaskFlow v6** is a suite of three custom widgets for [Grist](https://www.getgrist.com/) (open-source database platform) providing project/task management views: Kanban, Gantt, and Calendar. The entire UI is in **French**.

Each widget is a **fully standalone, self-contained HTML file** (~2000 lines each) with inline CSS and JavaScript. There is no build system, no package manager, and no framework — just vanilla JS, HTML5, and CSS3. Any widget can be used independently. When multiple widgets are placed in the same Grist document, they work in concert automatically — Grist acts as the communication bus (shared data via `onRecords`, shared selection via `setSelectedRows`/`onRecord`). The widgets never communicate directly with each other.

## Development

### Running Locally
Open any `.html` file directly in a browser — widgets auto-detect the absence of Grist and launch in **demo mode** with generated sample data. No server required.

### Deploying to Grist
Add as a Custom Widget in Grist using the raw GitHub URL (e.g., `https://raw.githubusercontent.com/USER/REPO/main/kanban.html`). All widgets must be linked to the `Tasks` table with "Select By" configured for cross-widget selection sync.

### External Dependencies
- **Grist Plugin API** — loaded via `<script src="https://docs.getgrist.com/grist-plugin-api.js">`
- **Sortable.js v1.15.0** — loaded via CDN for drag-and-drop

## Architecture

### Widget Structure (each file follows this pattern)
1. `<style>` block with CSS variables in `:root` for theming
2. HTML structure: header, main content area, hidden modals, toast container
3. `<script>` block organized in sections separated by `// ═══════` comments:
   - Global state (`let tasks = [], team = [], selectedTaskId = null`)
   - Utility functions (`escapeHtml`, `gristToDate`, `dateToGrist`, `formatDate`, `getAssigneesArray`)
   - Rendering functions
   - Modal open/close/save functions
   - Grist integration (`initGrist`, `loadAllData`)

### Grist API Integration Pattern
All three widgets use the same integration pattern:
```javascript
grist.ready({ requiredAccess: 'full' });
grist.onRecords(async (data) => { /* columnar → row conversion, then render */ });
grist.onRecord((record) => { /* sync selection from other widgets */ });
grist.setSelectedRows([taskId]);  // broadcast selection
grist.docApi.applyUserActions([['UpdateRecord', 'Tasks', id, { field: value }]]);
grist.docApi.fetchTable('Team');  // load secondary tables
```

### Data Conversion
Grist sends data in **columnar format**. Each widget has a `convert()` function that transforms it into an array of row objects. Dates are Unix timestamps converted via `gristToDate(ts)` (multiply by 1000 for JS Date) and `dateToGrist(date)` (divide by 1000).

### Cross-Widget Selection Sync
When a task is clicked in any widget, it calls `grist.setSelectedRows([id])`. Other widgets receive the change via `grist.onRecord()` and highlight the corresponding task. This is the core mechanism tying the three views together.

### Auto-Schema Creation
Widgets define a `TASKFLOW_SCHEMA` object and call `ensureSchema()` on init to auto-create missing tables/columns in Grist, with graceful fallback if they already exist.

## Data Schema

### Tasks table (required)
Key columns: `titre`, `description`, `dateDebut`, `dateEcheance`, `priorite` (1-4), `statut` (todo/inprogress/review/done), `progression` (0-100), `projet` (Ref→Projects), `assignees` (RefList→Team), `type` (tache/jalon/reunion), `dependDe` (RefList→Tasks), `tags`, `estimationH`, `tempsPasse`, `couleur`.

### Team table (optional)
Columns: `nom`, `email`, `avatar`, `role`, `actif`.

### Projects table (optional)
Columns: `nom`, `couleur`, `dateDebut`, `dateFin`, `responsable` (Ref→Team), `actif`.

## Common Task Management Structure

The functional core is **identical** across all 3 widgets. Any modification to a shared pattern must be propagated to all 3 files.

### Complete Lifecycle

```
1. initGrist()          → grist.ready({ requiredAccess: 'full' })
2. ensureSchema()       → Creates missing tables/columns via AddTable/AddColumn
3. loadAllData()        → fetchTable('Tasks'), fetchTable('Team'), fetchTable('Projects')
4. convert(data)        → Transforms Grist columnar format → JS object array
5. render()             → Widget-specific rendering
6. grist.onRecords()    → Reloads loadAllData() on each external change
7. grist.onRecord()     → Selection sync from another widget
```

If Grist is unavailable, `initGrist()` catches the error and switches to **demo mode** with generated data.

### Task CRUD (identical across all 3 widgets)

```javascript
// CREATE
await grist.docApi.applyUserActions([['AddRecord', 'Tasks', null, taskData]]);

// UPDATE
await grist.docApi.applyUserActions([['UpdateRecord', 'Tasks', parseInt(taskId), taskData]]);

// taskData format for create/update:
{
    titre, description, statut, priorite, type, progression,
    projet: projectId || null,
    dateDebut: dateToGrist(startDate),
    dateEcheance: dateToGrist(endDate),
    assignees: ids.length > 0 ? ['L', ...ids] : null  // RefList format
}
```

### Data Conversion (identical)

```javascript
// Grist columnar → object array (named convertGristToRecords in kanban, convert in gantt/calendar)
{ titre: ['A','B'], statut: ['todo','done'] }  →  [{ titre:'A', statut:'todo' }, { titre:'B', statut:'done' }]

// Dates: Unix timestamps ↔ JS Date
gristToDate(ts)   → new Date(ts * 1000)
dateToGrist(date)  → Math.floor(date.getTime() / 1000)

// Assignees: Grist RefList → ID array
getAssigneesArray(task)  → ['L', 1, 3]  becomes  [1, 3]
```

### Cross-Widget Selection (identical)

```javascript
// Send: on task click
selectedTaskId = taskId;
grist.setSelectedRows([taskId]);

// Receive: from another widget
grist.onRecord((record) => {
    if (record?.id && record.id !== selectedTaskId) {
        selectedTaskId = record.id;
        // highlight in DOM
    }
});
```

### Filters (same logic, different localStorage keys)

```
taskflow_kanban_filters  → { project, priority } + text searchQuery
taskflow_gantt_filters   → { project, assignee }
taskflow_calendar_filters → { project, priority, type }
```

Each widget filters via `getFilteredTasks()` or `taskMatchesFilters()` before rendering.

### Modals (same pattern)

- Open: `document.getElementById('taskModal').classList.add('open')`
- Close: `.classList.remove('open')`
- Create vs edit: distinguished by the presence or absence of a `taskId` in the hidden field
- Save: calls `AddRecord` or `UpdateRecord` as appropriate, then `showToast()` for confirmation

### Auto-schema (identical, `TASKFLOW_SCHEMA`)

All 3 widgets embed the same `TASKFLOW_SCHEMA` object defining the 3 tables (Tasks, Team, Projects) with all their column types. On first launch, `ensureSchema()`:
1. Attempts `fetchTable()` to check if the table exists
2. If not → `AddTable` with all columns
3. If yes → compares existing columns and adds missing ones via `AddColumn`
4. If the Tasks table was just created → `seedData()` injects sample data

### Priority Color Mapping
```
1 (Critical) → --danger (#ef4444)
2 (High)     → --warning (#f59e0b)
3 (Medium)   → --info (#3b82f6)
4 (Low)      → --text-muted (#64748b)
```
