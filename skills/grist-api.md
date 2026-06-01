# Grist API - Basic Patterns

## Initialization

### Simple widget (read-only)
```javascript
grist.ready({ requiredAccess: 'read table' });
```

### Widget with write access
```javascript
grist.ready({ requiredAccess: 'full' });
```

### Access levels
| Level | Capabilities |
|--------|-----------|
| `none` | No data access |
| `read table` | Read-only access to the linked table |
| `full` | Read/write access to the entire document |

---

## Reading Data

### onRecords — Data from the linked table
```javascript
grist.onRecords((data, mappings) => {
    // data = columnar format { id: [...], titre: [...], ... }
    // mappings = configured column mapping
    const records = convertToRows(data);
    render(records);
});
```

### fetchTable — Direct table read
```javascript
async function loadTable(tableName) {
    const data = await grist.docApi.fetchTable(tableName);
    return convertToRows(data);
}

// Examples
const tasks = await loadTable('Tasks');
const team = await loadTable('Team');
```

### listTables — List of document tables
```javascript
const tables = await grist.docApi.listTables();
// ['Tasks', 'Team', 'Projects', ...]
```

---

## Writing Data

### AddRecord — Create a record
```javascript
await grist.docApi.applyUserActions([
    ['AddRecord', 'Tasks', null, {
        titre: 'Nouvelle tâche',
        statut: 'todo',
        priorite: 3,
        dateDebut: Math.floor(Date.now() / 1000)
    }]
]);
```

### UpdateRecord — Modify a record
```javascript
await grist.docApi.applyUserActions([
    ['UpdateRecord', 'Tasks', taskId, {
        statut: 'done',
        progression: 100
    }]
]);
```

### RemoveRecord — Delete a record
```javascript
await grist.docApi.applyUserActions([
    ['RemoveRecord', 'Tasks', taskId]
]);
```

### Multiple actions
```javascript
await grist.docApi.applyUserActions([
    ['UpdateRecord', 'Tasks', 1, { statut: 'done' }],
    ['UpdateRecord', 'Tasks', 2, { statut: 'done' }],
    ['AddRecord', 'Tasks', null, { titre: 'Nouvelle' }]
]);
```

---

## Schema Creation

**See [schema.md](schema.md)** for full documentation:
- Table creation order (refs)
- Labels, widgetOptions, choices
- Ref/RefList configuration
- Calculated columns (formulas)
- Complete `ensureSchema()` pattern

---

## Demo mode (fallback without Grist)

```javascript
let isDemo = false;

async function init() {
    try {
        grist.ready({ requiredAccess: 'full' });
        await loadData();
    } catch (e) {
        console.log('Demo mode activated');
        isDemo = true;
        loadDemoData();
    }
}

function loadDemoData() {
    return [
        { id: 1, titre: 'Tâche exemple 1', statut: 'todo' },
        { id: 2, titre: 'Tâche exemple 2', statut: 'done' }
    ];
}
```
