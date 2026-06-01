# Architecture Patterns

## Widget HTML Structure

### Base template
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My Widget</title>
    <script src="https://docs.getgrist.com/grist-plugin-api.js"></script>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body {
            font-family: system-ui, -apple-system, sans-serif;
            background: transparent;
        }
        /* ... styles ... */
    </style>
</head>
<body>
    <div id="app">Loading...</div>

    <!-- Modals (optional) -->
    <div id="modal" class="modal"></div>

    <!-- Toast (optional) -->
    <div id="toast-container"></div>

    <script>
        // Widget code
    </script>
</body>
</html>
```

### JavaScript Organization
```javascript
// ═══════════════════════════════════════════════════════════
// GLOBAL STATE
// ═══════════════════════════════════════════════════════════
let records = [];
let selectedId = null;
let isDemo = false;

// ═══════════════════════════════════════════════════════════
// UTILITIES
// ═══════════════════════════════════════════════════════════
function escapeHtml(text) { /* ... */ }
function formatDate(ts) { /* ... */ }
function convertToRows(data) { /* ... */ }

// ═══════════════════════════════════════════════════════════
// RENDERING
// ═══════════════════════════════════════════════════════════
function render() { /* ... */ }

// ═══════════════════════════════════════════════════════════
// MODALS
// ═══════════════════════════════════════════════════════════
function openModal(id) { /* ... */ }
function closeModal() { /* ... */ }
function saveModal() { /* ... */ }

// ═══════════════════════════════════════════════════════════
// GRIST
// ═══════════════════════════════════════════════════════════
async function init() { /* ... */ }

// Startup
init();
```

---

## Modals

### HTML Structure
```html
<div id="taskModal" class="modal">
    <div class="modal-backdrop"></div>
    <div class="modal-content">
        <div class="modal-header">
            <h2 id="modalTitle">Title</h2>
            <button onclick="closeModal()" class="close-btn">&times;</button>
        </div>
        <div class="modal-body">
            <input type="hidden" id="taskId">
            <label>Title</label>
            <input type="text" id="titre">
            <!-- other fields -->
        </div>
        <div class="modal-footer">
            <button onclick="closeModal()" class="btn-secondary">Cancel</button>
            <button onclick="saveTask()" class="btn-primary">Save</button>
        </div>
    </div>
</div>
```

### Modal CSS
```css
.modal {
    display: none;
    position: fixed;
    inset: 0;
    z-index: 1000;
}
.modal.open { display: flex; align-items: center; justify-content: center; }

.modal-backdrop {
    position: absolute;
    inset: 0;
    background: rgba(0,0,0,0.5);
}

.modal-content {
    position: relative;
    background: white;
    border-radius: 12px;
    max-width: 500px;
    width: 90%;
    max-height: 90vh;
    overflow-y: auto;
    box-shadow: 0 25px 50px rgba(0,0,0,0.25);
}

.modal-header { padding: 20px; border-bottom: 1px solid #e5e7eb; }
.modal-body { padding: 20px; }
.modal-footer { padding: 20px; border-top: 1px solid #e5e7eb; display: flex; gap: 12px; justify-content: flex-end; }
```

### Modal JavaScript
```javascript
function openModal(taskId = null) {
    const modal = document.getElementById('taskModal');
    const title = document.getElementById('modalTitle');

    if (taskId) {
        // Edit mode
        const task = records.find(t => t.id === taskId);
        title.textContent = 'Edit task';
        document.getElementById('taskId').value = taskId;
        document.getElementById('titre').value = task.titre || '';
        // ... fill other fields
    } else {
        // Create mode
        title.textContent = 'New task';
        document.getElementById('taskId').value = '';
        document.getElementById('titre').value = '';
        // ... clear other fields
    }

    modal.classList.add('open');
}

function closeModal() {
    document.getElementById('taskModal').classList.remove('open');
}

async function saveTask() {
    const taskId = document.getElementById('taskId').value;
    const data = {
        titre: document.getElementById('titre').value,
        // ... other fields
    };

    try {
        if (taskId) {
            await grist.docApi.applyUserActions([
                ['UpdateRecord', 'Tasks', parseInt(taskId), data]
            ]);
            showToast('Task updated');
        } else {
            await grist.docApi.applyUserActions([
                ['AddRecord', 'Tasks', null, data]
            ]);
            showToast('Task created');
        }
        closeModal();
    } catch (e) {
        showToast('Error: ' + e.message, 'error');
    }
}
```

---

## Toast / Notifications

### HTML
```html
<div id="toast-container"></div>
```

### CSS
```css
#toast-container {
    position: fixed;
    bottom: 20px;
    right: 20px;
    z-index: 9999;
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.toast {
    padding: 12px 20px;
    border-radius: 8px;
    background: #1f2937;
    color: white;
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
    animation: slideIn 0.3s ease;
}

.toast.success { background: #10b981; }
.toast.error { background: #ef4444; }
.toast.warning { background: #f59e0b; }

@keyframes slideIn {
    from { transform: translateX(100%); opacity: 0; }
    to { transform: translateX(0); opacity: 1; }
}
```

### JavaScript
```javascript
function showToast(message, type = 'success', duration = 3000) {
    const container = document.getElementById('toast-container');
    const toast = document.createElement('div');
    toast.className = `toast ${type}`;
    toast.textContent = message;

    container.appendChild(toast);

    setTimeout(() => {
        toast.style.animation = 'slideIn 0.3s ease reverse';
        setTimeout(() => toast.remove(), 300);
    }, duration);
}
```

---

## Filters

### localStorage storage
```javascript
const FILTER_KEY = 'mywidget_filters';

function loadFilters() {
    try {
        return JSON.parse(localStorage.getItem(FILTER_KEY)) || {};
    } catch {
        return {};
    }
}

function saveFilters(filters) {
    localStorage.setItem(FILTER_KEY, JSON.stringify(filters));
}
```

### Applying filters
```javascript
let filters = loadFilters();

function getFilteredRecords() {
    return records.filter(record => {
        if (filters.project && record.projet !== filters.project) return false;
        if (filters.priority && record.priorite !== filters.priority) return false;
        if (filters.search) {
            const search = filters.search.toLowerCase();
            if (!record.titre?.toLowerCase().includes(search)) return false;
        }
        return true;
    });
}

function setFilter(key, value) {
    if (value) {
        filters[key] = value;
    } else {
        delete filters[key];
    }
    saveFilters(filters);
    render();
}
```

### Filter UI
```html
<div class="filters">
    <input type="text" id="search" placeholder="Search..."
           oninput="setFilter('search', this.value)">

    <select id="projectFilter" onchange="setFilter('project', this.value)">
        <option value="">All projects</option>
        <!-- dynamic options -->
    </select>

    <select id="priorityFilter" onchange="setFilter('priority', this.value)">
        <option value="">All priorities</option>
        <option value="1">Critical</option>
        <option value="2">High</option>
        <option value="3">Medium</option>
        <option value="4">Low</option>
    </select>
</div>
```

---

## CSS Variables (theming)

```css
:root {
    /* Primary colors */
    --primary: #3b82f6;
    --primary-dark: #2563eb;
    --success: #10b981;
    --warning: #f59e0b;
    --danger: #ef4444;
    --info: #06b6d4;

    /* Text */
    --text: #1f2937;
    --text-muted: #6b7280;
    --text-light: #9ca3af;

    /* Background */
    --bg: #ffffff;
    --bg-secondary: #f3f4f6;
    --bg-hover: #f9fafb;

    /* Borders */
    --border: #e5e7eb;
    --border-dark: #d1d5db;

    /* Shadows */
    --shadow-sm: 0 1px 2px rgba(0,0,0,0.05);
    --shadow: 0 4px 6px rgba(0,0,0,0.1);
    --shadow-lg: 0 10px 25px rgba(0,0,0,0.15);

    /* Radii */
    --radius-sm: 4px;
    --radius: 8px;
    --radius-lg: 12px;

    /* Transitions */
    --transition: 0.2s ease;
}
```

### Usage
```css
.card {
    background: var(--bg);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    box-shadow: var(--shadow-sm);
    transition: box-shadow var(--transition);
}

.card:hover {
    box-shadow: var(--shadow);
}

.btn-primary {
    background: var(--primary);
    color: white;
}

.btn-primary:hover {
    background: var(--primary-dark);
}
```

---

## Priority Mapping

```javascript
const PRIORITY_CONFIG = {
    1: { label: 'Critical', color: '#ef4444', icon: '🔴' },
    2: { label: 'High', color: '#f59e0b', icon: '🟠' },
    3: { label: 'Medium', color: '#3b82f6', icon: '🔵' },
    4: { label: 'Low', color: '#6b7280', icon: '⚪' }
};

function getPriorityBadge(priority) {
    const config = PRIORITY_CONFIG[priority] || PRIORITY_CONFIG[3];
    return `<span class="priority-badge" style="background:${config.color}">${config.label}</span>`;
}
```

---

## Status Mapping

```javascript
const STATUS_CONFIG = {
    todo: { label: 'To do', color: '#6b7280', icon: '📋' },
    inprogress: { label: 'In progress', color: '#3b82f6', icon: '🔄' },
    review: { label: 'Review', color: '#f59e0b', icon: '👀' },
    done: { label: 'Done', color: '#10b981', icon: '✅' }
};
```

---

## Initial Loading

```javascript
async function init() {
    showLoading(true);

    try {
        grist.ready({ requiredAccess: 'full' });

        // Load data
        await loadAllData();

        // Listen for changes
        grist.onRecords(handleRecordsUpdate);
        grist.onRecord(handleRecordSelect);

    } catch (e) {
        console.log('Demo mode:', e.message);
        isDemo = true;
        loadDemoData();
    }

    showLoading(false);
    render();
}

function showLoading(show) {
    document.getElementById('app').innerHTML = show
        ? '<div class="loading">Loading...</div>'
        : '';
}

// Start
init();
```
