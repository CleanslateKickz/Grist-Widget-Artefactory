# TaskFlow v6 - Improvement Specifications

## Main Objectives

### 1. Shared Grist Architecture
- Cross-widget selection synchronization on the same page
- Centralized column configuration via widget options
- Optimal and documented table schema
- Standalone mode + integrated mode

### 2. Per-Widget Improvements

---

## Optimal Grist Table Schema

### `Tasks` Table (main)
```
| Column       | Type                  | Description                    | Required    |
|---------------|----------------------|--------------------------------|-------------|
| id            | Integer (auto)        | Unique ID                      | ✓           |
| titre         | Text                  | Task name                      | ✓           |
| description   | Text (long)           | Detailed description            |             |
| dateDebut     | Date                  | Start date                     | ✓           |
| dateEcheance  | Date                  | Due date                       | ✓           |
| priorite      | Choice (1,2,3,4)      | 1=Critical, 4=Low              | ✓           |
| statut        | Choice                | To do, In progress, In review, Done | ✓     |
| progression   | Integer (0-100)       | Completion percentage          |             |
| projet        | Reference (Projects)  | Link to project                 |             |
| assignees     | Reference List (Team) | Assigned members                |             |
| type          | Choice                | tache, jalon, reunion           |             |
| dependDe      | Reference List (Tasks)| Predecessor tasks               |             |
| tags          | Choice List           | Tags                            |             |
| estimationH   | Numeric               | Estimated hours                |             |
| tempsPasse    | Numeric               | Hours spent                     |             |
| couleur       | Text                  | Custom color (#hex)             |             |
```

### `Team` Table (team)
```
| Column       | Type           | Description              |
|---------------|----------------|--------------------------|
| id            | Integer (auto) | Unique ID                |
| nom           | Text           | Full name                |
| email         | Text           | Email                    |
| avatar        | Text           | URL or initials          |
| role          | Choice         | Role in the team         |
| actif         | Bool           | Active member            |
```

### `Projects` Table (projects)
```
| Column       | Type           | Description              |
|---------------|----------------|--------------------------|
| id            | Integer (auto) | Unique ID                |
| nom           | Text           | Project name             |
| couleur       | Text           | Color (#hex)             |
| dateDebut     | Date           | Project start date       |
| dateFin       | Date           | Expected end date        |
| responsable   | Reference(Team)| Project manager          |
| actif         | Bool           | Active project           |
```

### `StatusConfig` Table (status configuration - for Kanban)
```
| Column       | Type           | Description              |
|---------------|----------------|--------------------------|
| id            | Integer (auto) | Unique ID                |
| nom           | Text           | Status name              |
| couleur       | Text           | Color (#hex)             |
| ordre         | Integer        | Position in kanban       |
| icone         | Text           | Emoji or icon            |
```

---

## Inter-Widget Grist Synchronization

### Shared selection mechanism
```javascript
// Each widget listens for selection changes
grist.onRecord((record) => {
    // Update local selection
    selectedTaskId = record?.id || null;
    highlightSelectedTask();
});

// Each widget notifies Grist of its selection
function selectTask(taskId) {
    selectedTaskId = taskId;
    grist.setSelectedRows([taskId]);
    highlightSelectedTask();
}
```

### Configuration via Widget Options
```javascript
grist.onOptions((options) => {
    // Kanban: category field
    config.groupByColumn = options?.groupByColumn || 'statut';
    
    // Gantt: display
    config.showDependencies = options?.showDependencies !== false;
    config.defaultView = options?.defaultView || 'month';
    
    // All: default filters
    config.defaultProject = options?.defaultProject || null;
});
```

---

## KANBAN - v6 Improvements

### 1. Configurable Columns
- **Grouping field**: Selectable (statut, projet, priorite, assigne, tag)
- **Add column**: Discrete "+" button to create a new value
- **Reorganization**: Drag & drop of entire columns

### 2. Create/Edit Modal
```
┌─────────────────────────────────────────────────┐
│ ✏️ New task                                [×] │
├─────────────────────────────────────────────────┤
│ Title *                                         │
│ [___________________________________________]   │
│                                                 │
│ Description                                     │
│ [___________________________________________]   │
│ [___________________________________________]   │
│                                                 │
│ ┌─────────────┐ ┌─────────────┐                │
│ │ Dates       │ │ Assignment  │                │
│ │ Start: [__] │ │ [Assignees ▼]│                │
│ │ End:   [__] │ │ [Project  ▼]│                │
│ └─────────────┘ └─────────────┘                │
│                                                 │
│ ┌─────────────┐ ┌─────────────┐                │
│ │ Priority    │ │ Type        │                │
│ │ [●○○○ Crit.]│ │ [Task     ▼]│                │
│ └─────────────┘ └─────────────┘                │
│                                                 │
│ Progress    [====____] 40%                      │
│                                                 │
│ Tags  [Backend] [+]                              │
│                                                 │
├─────────────────────────────────────────────────┤
│                      [Cancel]  [💾 Save]        │
└─────────────────────────────────────────────────┘
```

### 3. Advanced Filters
- Text search (title, description)
- Combinable multi-filters (project + priority + assignee)
- Save favorite filters
- Visual indicator of active filters

### 4. Optimized Task Card
```
┌────────────────────────────────────┐
│ │ Task title                      │ ← Priority bar
│ │                                 │
│ │ [Critical] [Backend]            │ ← Badges
│ │                                 │
│ │ 📅 Feb 14 ━━━━━━ 60%    👤👤   │ ← Date, progress, avatars
│ │                                 │
│ │ ◆ Milestone (if applicable)     │
└────────────────────────────────────┘
```

---

## GANTT - v6 Improvements

### 1. Continuous Scroll (Infinite Scroll)
- No fixed "period" — smooth navigation
- Dynamic cell loading
- Optimized performance (virtual scrolling)

### 2. Improved Visual Dependencies
```
Link types:
- Finish → Start (FS): Standard
- Start → Start (SS)
- Finish → Finish (FF)
- Start → Finish (SF)

Display:
- Bezier curved lines
- Directional arrows
- Hover highlighting
- Click to select the link
```

### 3. Distinct Milestones
- More visible diamond
- Label always displayed
- Differentiated icon (★ for release, ◆ for review, etc.)

### 4. Complete Gantt Modal
```
┌─────────────────────────────────────────────────────┐
│ 📊 Task details                               [×]  │
├─────────────────────────────────────────────────────┤
│ [Dev Frontend]                                      │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 40%                  │
│                                                     │
│ ┌─────────────────┬─────────────────┐              │
│ │ 📅 Dates        │ 👥 Team         │              │
│ │ Start: Feb 04   │ Alice Martin    │              │
│ │ End:   Feb 19   │ Bob Durant      │              │
│ │ Duration: 15 days│                 │              │
│ └─────────────────┴─────────────────┘              │
│                                                     │
│ 🔗 Dependencies                                     │
│ ← Design UI (must finish first)                     │
│ → QA Tests (waiting on this task)                   │
│                                                     │
│ 📝 Notes                                            │
│ [_____________________________________________]     │
│                                                     │
├─────────────────────────────────────────────────────┤
│ [Edit in Grist]                  [Close]            │
└─────────────────────────────────────────────────────┘
```

### 5. Optimization Suggestions
- ⚠️ Overdue task (due date passed, progress < 100%)
- 🔄 Resource conflict (same assignee, same period)
- 📊 Critical path (tasks impacting end date)

---

## CALENDAR - v6 Improvements

### 1. Event Types
- **Task**: With duration (bar spanning multiple days)
- **Milestone**: Point event (diamond)
- **Meeting**: Time slot (for week view)

### 2. Quick Creation Modal
```
┌─────────────────────────────────────────┐
│ + New event                         [×]  │
├─────────────────────────────────────────┤
│ Type: [● Task] [○ Milestone] [○ Meeting]│
│                                         │
│ Title *                                 │
│ [________________________________]      │
│                                         │
│ 📅 February 4, 2026                      │
│                                         │
│ [+ Add end time]                        │
│                                         │
│ [See more options...]                   │
├─────────────────────────────────────────┤
│                    [Cancel] [Create]    │
└─────────────────────────────────────────┘
```

### 3. Week View with Hours
```
         Mon 3    Tue 4    Wed 5    Thu 6    Fri 7
       ┌────────┬────────┬────────┬────────┬────────┐
 08:00 │        │        │        │        │        │
 09:00 │ ██████ │        │        │        │        │ ← Meeting
 10:00 │        │        │ ██████████████████████  │ ← Multi-day task
 11:00 │        │        │        │        │        │
 ...   │        │        │ ◆      │        │        │ ← Milestone
```

### 4. Optimized Navigation
- Mobile swipe (left/right)
- Keyboard shortcuts (←/→, T for today)
- Mini-calendar for quick navigation

---

## Technical Implementation

### Implementation Order
1. **Shared core**: Schema, Grist sync, common modals
2. **Kanban v6**: Dynamic columns, quick creation
3. **Gantt v6**: Infinite scroll, dependencies
4. **Calendar v6**: Modals, week view

### Files to Create
```
/home/claude/taskflow/v6/
├── shared/
│   ├── grist-sync.js      # Grist synchronization
│   ├── modal-system.js    # Modal system
│   └── styles-common.css  # Shared styles
├── kanban.html
├── gantt.html
├── calendar.html
└── dashboard.html
```

### Performance
- Virtual scrolling for large lists
- Debounce on scroll/resize events
- Lazy data loading
- Local cache (localStorage)

---

## Deliverables

### Phase 1: Core + Kanban (high priority)
- [ ] Unified modal system
- [ ] Kanban with configurable columns
- [ ] Task creation from Kanban

### Phase 2: Advanced Gantt
- [ ] Continuous scroll
- [ ] Visual dependencies
- [ ] Complete modal

### Phase 3: Complete Calendar
- [ ] Month + week views with hours
- [ ] Quick event creation
- [ ] Drag & drop to move
