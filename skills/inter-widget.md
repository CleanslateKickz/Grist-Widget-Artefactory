# Inter-Widget Communication

## Overview

Grist allows multiple widgets to communicate via:
1. **Shared selection** — `setSelectedRows` / `onRecord`
2. **Shared cursor** — `setCursorPos` / `onRecord`
3. **Widget options** — `setOption` / `onOptions`
4. **Shared data** — The Grist table acts as the communication "bus"

---

## Shared Selection

### Emitting a selection
```javascript
let selectedTaskId = null;

function selectTask(taskId) {
    selectedTaskId = taskId;
    grist.setSelectedRows([taskId]);
    highlightTask(taskId);
}

// On task click
document.addEventListener('click', (e) => {
    const taskEl = e.target.closest('[data-task-id]');
    if (taskEl) {
        selectTask(parseInt(taskEl.dataset.taskId));
    }
});
```

### Receiving a selection (from another widget)
```javascript
grist.onRecord((record) => {
    if (!record) return;

    // Avoid infinite loop
    if (record.id === selectedTaskId) return;

    selectedTaskId = record.id;
    highlightTask(record.id);
    scrollToTask(record.id);
});
```

### Complete pattern (TaskFlow)
```javascript
// Global state
let selectedTaskId = null;

// Initialization
grist.ready({ requiredAccess: 'full' });

// Receive external selection
grist.onRecord((record) => {
    if (record?.id && record.id !== selectedTaskId) {
        selectedTaskId = record.id;

        // Visual highlight
        document.querySelectorAll('.task').forEach(el => {
            el.classList.toggle('selected',
                parseInt(el.dataset.taskId) === selectedTaskId
            );
        });
    }
});

// Emit on click
function onTaskClick(taskId) {
    selectedTaskId = taskId;
    grist.setSelectedRows([taskId]);
}
```

---

## Shared Cursor (navigation)

### "Select By" configuration
In Grist, configure the widget with "Select By" to receive the cursor from another widget.

### Navigation pattern (grist-navigation-widgets)
```javascript
// Navigation Widget - Emit selected page
async function selectPage(pageId) {
    await grist.setCursorPos({ rowId: pageId });
}

// Content Widget - Receive and filter
grist.onRecord((record) => {
    if (record) {
        currentPageId = record.id;
        filterContentByPage(currentPageId);
    }
});
```

---

## Widget Options

### Storing persistent options
```javascript
// Save an option (persists with the widget)
await grist.setOption('theme', 'dark');
await grist.setOption('filters', { project: 1, priority: 2 });

// Retrieve options
grist.onOptions((options) => {
    if (options?.theme) applyTheme(options.theme);
    if (options?.filters) applyFilters(options.filters);
});
```

### Complete pattern
```javascript
let currentOptions = {};

grist.onOptions((options) => {
    currentOptions = options || {};
    applyOptions(currentOptions);
});

async function saveOption(key, value) {
    currentOptions[key] = value;
    await grist.setOption(key, value);
}
```

---

## Pattern: Synchronized Multi-Widgets

### Architecture (3 TaskFlow widgets)
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Kanban    │     │    Gantt    │     │  Calendar   │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       ▼                   ▼                   ▼
    setSelectedRows     onRecord           onRecord
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────▼──────┐
                    │    Grist    │
                    │  (Tasks)    │
                    └─────────────┘
```

### Identical code in each widget
```javascript
// Each widget uses this same pattern
let selectedTaskId = null;

// Emit (on click)
function selectTask(taskId) {
    if (taskId === selectedTaskId) return;
    selectedTaskId = taskId;
    grist.setSelectedRows([taskId]);
    highlightInDOM(taskId);
}

// Receive (from another widget)
grist.onRecord((record) => {
    if (record?.id && record.id !== selectedTaskId) {
        selectedTaskId = record.id;
        highlightInDOM(record.id);
    }
});
```

---

## Pattern: Navigation + Content (grist-navigation-widgets)

### Navigation Widget
```javascript
// Pages table structure
// { id, nom, icone, ordre }

let pages = [];
let currentPageId = null;

grist.onRecords((data) => {
    pages = convertToRows(data);
    renderNavigation();
});

async function navigateTo(pageId) {
    currentPageId = pageId;
    await grist.setCursorPos({ rowId: pageId });
    highlightCurrentPage();
}
```

### Content Widget
```javascript
// Content table structure
// { id, page (Ref:Pages), titre, contenu }

let allContent = [];
let filteredContent = [];

grist.onRecords((data) => {
    allContent = convertToRows(data);
    filterByCurrentPage();
});

grist.onRecord((record) => {
    // record = the page selected in Navigation
    if (record) {
        currentPageId = record.id;
        filterByCurrentPage();
    }
});

function filterByCurrentPage() {
    filteredContent = allContent.filter(c => c.page === currentPageId);
    render();
}
```

---

## Pattern: App Runtime (Artefactory)

### window.app object
```javascript
window.app = {
    // Navigation between components
    navigate(path) {
        window.parent.postMessage({
            type: 'app-navigate',
            path
        }, '*');
    },

    // Inter-component events
    emit(eventName, data) {
        window.parent.postMessage({
            type: 'app-event',
            event: eventName,
            data
        }, '*');
    },

    on(eventName, callback) {
        window.addEventListener('message', (e) => {
            if (e.data?.type === 'app-event' && e.data.event === eventName) {
                callback(e.data.data);
            }
        });
    },

    // Shared state
    state: {},

    // App manifest
    manifest: null
};
```

### Usage in components
```javascript
// Component A: emit an event
function onBatimentSelect(batId) {
    appContext.app.emit('batiment-selected', { id: batId });
}

// Component B: receive the event
appContext.app.on('batiment-selected', (data) => {
    loadLocaux(data.id);
});

// Navigation
document.getElementById('btnLocaux').onclick = () => {
    appContext.app.navigate('/locaux');
};
```

---

## Context Detection

### Standalone vs integrated widget
```javascript
const appContext = {
    // Detect if running inside Artefactory
    isArtefactory: typeof window.app !== 'undefined' &&
                   typeof window.app.navigate === 'function',

    // API with fallback
    app: window.app || {
        navigate: () => console.log('Navigation not available'),
        emit: () => {},
        on: () => {},
        state: {}
    }
};

// Usage
if (appContext.isArtefactory) {
    appContext.app.emit('ready', { component: 'Dashboard' });
}
```
