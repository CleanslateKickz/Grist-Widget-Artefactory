# Grist Data Conversion

## Columnar Format → Objects

Grist sends data in **columnar format**:
```javascript
// Grist format (columnar)
{
    id: [1, 2, 3],
    titre: ['Tâche A', 'Tâche B', 'Tâche C'],
    statut: ['todo', 'done', 'todo']
}
```

Widgets work with **objects**:
```javascript
// Widget format (objects)
[
    { id: 1, titre: 'Tâche A', statut: 'todo' },
    { id: 2, titre: 'Tâche B', statut: 'done' },
    { id: 3, titre: 'Tâche C', statut: 'todo' }
]
```

---

## Conversion Functions

### Simple version
```javascript
function convertToRows(data) {
    const n = data.id?.length || 0;
    const records = [];
    for (let i = 0; i < n; i++) {
        const record = {};
        Object.keys(data).forEach(key => {
            record[key] = data[key][i];
        });
        records.push(record);
    }
    return records;
}
```

### Version with error handling
```javascript
function safeConvert(data) {
    if (!data || !data.id) return [];
    const n = data.id.length;
    const records = [];
    for (let i = 0; i < n; i++) {
        const record = { id: data.id[i] };
        Object.keys(data).forEach(key => {
            if (key !== 'id') {
                record[key] = data[key]?.[i] ?? null;
            }
        });
        records.push(record);
    }
    return records;
}
```

### Version with column mappings
```javascript
function convertWithMappings(data, mappings) {
    const n = data.id?.length || 0;
    const records = [];
    for (let i = 0; i < n; i++) {
        const record = { id: data.id[i] };
        // Apply mappings configured in Grist
        Object.entries(mappings).forEach(([widgetCol, gristCol]) => {
            record[widgetCol] = data[gristCol]?.[i];
        });
        records.push(record);
    }
    return records;
}
```

---

## Dates

### Grist → JavaScript
Grist stores dates as **Unix timestamps (seconds)**, JavaScript uses **milliseconds**.

```javascript
function gristToDate(timestamp) {
    if (!timestamp) return null;
    return new Date(timestamp * 1000);
}

function gristToISOString(timestamp) {
    if (!timestamp) return '';
    return new Date(timestamp * 1000).toISOString().split('T')[0];
}
```

### JavaScript → Grist
```javascript
function dateToGrist(date) {
    if (!date) return null;
    if (typeof date === 'string') date = new Date(date);
    return Math.floor(date.getTime() / 1000);
}

// Examples
dateToGrist(new Date())           // Now
dateToGrist('2024-03-15')         // ISO date
dateToGrist(Date.now())           // JS timestamp (ms) → Grist (s)
```

### Formatting for display
```javascript
function formatDate(timestamp, locale = 'fr-FR') {
    if (!timestamp) return '';
    return new Date(timestamp * 1000).toLocaleDateString(locale);
}

function formatDateTime(timestamp, locale = 'fr-FR') {
    if (!timestamp) return '';
    return new Date(timestamp * 1000).toLocaleString(locale);
}
```

---

## RefList (multiple references)

### Grist format
```javascript
// RefList in Grist = ['L', id1, id2, id3]
// The 'L' indicates a list
{ assignees: ['L', 1, 3, 5] }
```

### Extracting IDs
```javascript
function getRefListIds(refList) {
    if (!refList) return [];
    if (!Array.isArray(refList)) return [];
    if (refList[0] === 'L') {
        return refList.slice(1);
    }
    return refList;
}

// Usage
const assigneeIds = getRefListIds(task.assignees);
// [1, 3, 5]
```

### Creating for write
```javascript
function createRefList(ids) {
    if (!ids || ids.length === 0) return null;
    return ['L', ...ids.map(id => parseInt(id))];
}

// Usage
const refList = createRefList([1, 3, 5]);
// ['L', 1, 3, 5]
```

---

## Ref (simple reference)

```javascript
// Simple reference = just the ID (number)
{ projet: 42 }

// Reading
const projectId = task.projet; // 42

// Writing
await grist.docApi.applyUserActions([
    ['UpdateRecord', 'Tasks', taskId, { projet: 42 }]
]);
```

---

## Reference Resolution

### Join linked data
```javascript
async function loadTasksWithTeam() {
    const [tasksData, teamData] = await Promise.all([
        grist.docApi.fetchTable('Tasks'),
        grist.docApi.fetchTable('Team')
    ]);

    const tasks = convertToRows(tasksData);
    const team = convertToRows(teamData);
    const teamById = Object.fromEntries(team.map(m => [m.id, m]));

    // Enrich tasks with team data
    return tasks.map(task => ({
        ...task,
        assigneesData: getRefListIds(task.assignees)
            .map(id => teamById[id])
            .filter(Boolean)
    }));
}
```

---

## HTML Escape (security)

```javascript
function escapeHtml(text) {
    if (!text) return '';
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
}

// Usage in templates
`<div class="title">${escapeHtml(task.titre)}</div>`
```
