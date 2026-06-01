# TaskFlow v6 - Initialization Guide & Grist Schema

## Overview

TaskFlow v6 is a suite of Grist widgets for project management, including:
- **Kanban**: Visual management with configurable columns
- **Gantt**: Timeline with dependencies and milestones
- **Calendar**: Calendar view with quick creation
- **Dashboard**: Summary view (coming soon)

## Grist Table Schema

### Main Table: `Tasks`

This table is **required** for the widgets to function.

```
┌─────────────────┬──────────────────────┬─────────────────────────────────────────┬───────────┐
│ Column          │ Grist Type           │ Description                             │ Required  │
├─────────────────┼──────────────────────┼─────────────────────────────────────────┼───────────┤
│ id              │ Integer (auto)       │ Unique ID (auto-generated)              │ ✓ auto    │
│ titre           │ Text                 │ Task name                               │ ✓         │
│ description     │ Text (long)          │ Detailed description                     │           │
│ dateDebut       │ Date                 │ Start date                              │ ✓         │
│ dateEcheance    │ Date                 │ Due date                                │ ✓         │
│ priorite        │ Choice               │ 1=Critical, 2=High, 3=Medium, 4=Low    │ ✓         │
│ statut          │ Choice               │ todo, inprogress, review, done           │ ✓         │
│ progression     │ Numeric (0-100)      │ Completion percentage                    │           │
│ projet          │ Reference (Projects) │ Link to Projects table                  │           │
│ assignees       │ Reference List (Team)│ Assigned members                        │           │
│ type            │ Choice               │ tache, jalon, reunion                   │           │
│ dependDe        │ Reference List (Tasks)│ Predecessor tasks                       │           │
│ tags            │ Choice List          │ Tags                                    │           │
│ estimationH     │ Numeric              │ Estimated hours                         │           │
│ tempsPasse      │ Numeric              │ Hours spent                             │           │
│ couleur         │ Text                 │ Custom color (#hex)                     │           │
└─────────────────┴──────────────────────┴─────────────────────────────────────────┴───────────┘
```

**Choice Configuration:**

```
priorite:
  - 1 (Critical)
  - 2 (High)
  - 3 (Medium)
  - 4 (Low)

statut:
  - todo (To do)
  - inprogress (In progress)
  - review (In review)
  - done (Done)

type:
  - tache (Standard task)
  - jalon (Milestone)
  - reunion (Meeting)
```

### Secondary Table: `Team` (Optional)

For avatar display and assignee filtering.

```
┌─────────────────┬──────────────────────┬─────────────────────────────────────────┬───────────┐
│ Column          │ Grist Type           │ Description                             │ Required  │
├─────────────────┼──────────────────────┼─────────────────────────────────────────┼───────────┤
│ id              │ Integer (auto)       │ Unique ID                               │ ✓ auto    │
│ nom             │ Text                 │ Full name                               │ ✓         │
│ email           │ Text                 │ Email                                   │           │
│ avatar          │ Text                 │ URL or initials                         │           │
│ role            │ Choice               │ Role in the team                        │           │
│ actif           │ Bool                 │ Active member                           │           │
└─────────────────┴──────────────────────┴─────────────────────────────────────────┴───────────┘
```

### Secondary Table: `Projects` (Optional)

For multi-project management and colors.

```
┌─────────────────┬──────────────────────┬─────────────────────────────────────────┬───────────┐
│ Column          │ Grist Type           │ Description                             │ Required  │
├─────────────────┼──────────────────────┼─────────────────────────────────────────┼───────────┤
│ id              │ Integer (auto)       │ Unique ID                               │ ✓ auto    │
│ nom             │ Text                 │ Project name                            │ ✓         │
│ couleur         │ Text                 │ Color (#hex)                            │           │
│ dateDebut       │ Date                 │ Project start date                      │           │
│ dateFin         │ Date                 │ Expected end date                       │           │
│ responsable     │ Reference (Team)     │ Project manager                         │           │
│ actif           │ Bool                 │ Active project                          │           │
└─────────────────┴──────────────────────┴─────────────────────────────────────────┴───────────┘
```

## Installation

### 1. Create the Tables

1. In Grist, create a new `Tasks` table
2. Add columns according to the schema above
3. (Optional) Create the `Team` and `Projects` tables

### 2. Add the Widgets

1. Click "Add New" → "Add widget to page"
2. Select "Custom" widget
3. In the widget configuration, enter the HTML file URL:
   - For GitHub: `https://raw.githubusercontent.com/YOUR_USER/grist-taskflow/main/kanban.html`
   - For local file: path to the file

### 3. Recommended Configuration

**Suggested Page Layout:**

```
┌────────────────────────────────────────────────────────────────────┐
│ [Table Tasks]                           [Kanban Custom Widget]     │
├────────────────────────────────────────────────────────────────────┤
│ [Gantt Custom Widget - Full Width]                                 │
├────────────────────────────────────────────────────────────────────┤
│ [Calendar Custom Widget]                [Dashboard Widget]         │
└────────────────────────────────────────────────────────────────────┘
```

### 4. Link the Widgets

For selection to be synchronized between widgets:
1. Ensure all widgets are linked to the `Tasks` table
2. In each widget's options, select "Select By" → Table Tasks
3. Clicks in one widget will automatically select the row in the others

## Grist Synchronization

### Selection Mechanism

The widgets use the Grist API to:
- **Receive data**: `grist.onRecords()` - called when data changes
- **Receive selection**: `grist.onRecord()` - called when a row is selected elsewhere
- **Send selection**: `grist.setSelectedRows([id])` - selects a row

### Synchronization Code (in each widget)

```javascript
// Initialization
grist.ready({ requiredAccess: 'full' });

// Data reception
grist.onRecords(async (data) => {
    tasks = convertGristToRecords(data);
    render();
});

// External selection reception
grist.onRecord((record) => {
    if (record?.id) {
        selectedTaskId = record.id;
        highlightSelectedTask();
    }
});

// Send selection
function selectTask(taskId) {
    selectedTaskId = taskId;
    grist.setSelectedRows([taskId]);
    highlightSelectedTask();
}
```

## Per-Widget Configuration

### Kanban

**Configurable grouping:**
- By `statut` (default): Columns todo, inprogress, review, done
- By `priorite`: Columns Critical, High, Medium, Low
- By `projet`: One column per project
- By `assignee`: One column per team member

**Add column:**
- "+" button at the end of columns (status mode only)
- Adds a new value to the `statut` Choice

### Gantt

**Configurable sorting:**
- By priority (default)
- By start date
- Manual (drag & drop)

**Dependencies:**
- `dependDe` column: Reference List to other tasks
- Display: Curved arrows between bars
- Supported types: Finish → Start

**Navigation:**
- Views: Week, Month, Quarter, Year
- Continuous horizontal scroll
- Previous/next buttons

### Calendar

**Views:**
- Month: Classic 7×6 grid
- Week: Hourly timeline

**Event types:**
- `tache`: Colored bar spanning multiple days
- `jalon`: Diamond with border
- `reunion`: Event with time

## Customization

### Priority Colors

```css
--danger: #ef4444;   /* Critical (1) */
--warning: #f59e0b;  /* High (2) */
--info: #3b82f6;     /* Medium (3) */
--text-muted: #64748b; /* Low (4) */
```

### Theme

Modify CSS variables in `:root` to customize:

```css
:root {
    --primary: #4f46e5;      /* Primary color */
    --primary-light: #e0e7ff; /* Selection background */
    --bg: #f8fafc;           /* Page background */
    --card-bg: #ffffff;      /* Card background */
    --text: #1e293b;         /* Main text */
    --border: #e2e8f0;       /* Borders */
}
```

## Demo Mode

The widgets work without Grist using demonstration data:
- "Demo" badge displayed
- Data generated automatically
- All features available (except persistence)

To force demo mode, open the HTML file directly in a browser.

## Files

```
taskflow/v6/
├── kanban.html      # Kanban widget
├── gantt.html       # Gantt widget
├── calendar.html    # Calendar widget
├── SPECIFICATIONS.md # Detailed specifications
└── README.md        # This file
```

## Installation Checklist

- [ ] `Tasks` table created with required columns
- [ ] Choices configured (priorite, statut, type)
- [ ] (Optional) `Team` table created
- [ ] (Optional) `Projects` table created
- [ ] Kanban widget added and linked to Tasks
- [ ] Gantt widget added and linked to Tasks
- [ ] Calendar widget added and linked to Tasks
- [ ] Cross-widget selection test

## Troubleshooting

**Data is not displaying:**
- Check that the widget is linked to the `Tasks` table
- Check column names (case-sensitive)
- Open the browser console (F12) for errors

**Selection is not synchronized:**
- Check "Select By" in the widget options
- Ensure all widgets point to `Tasks`

**Widget shows "Demo":**
- The widget is not properly integrated with Grist
- Check the widget URL in the configuration

## Changelog

### v6.0.0
- Configurable Kanban columns (status/priority/project/assignee)
- Task creation with contextual modal
- Gantt with continuous scroll
- Improved visual dependencies
- Calendar with click-to-quick-create
- Detailed modals for all widgets
- Advanced filters with localStorage persistence
- Optimized Grist synchronization
