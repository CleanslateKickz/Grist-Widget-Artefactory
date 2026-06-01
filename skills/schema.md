# Grist Schema Initialization

## Creation Order

**CRITICAL**: Referenced tables must exist BEFORE the tables that reference them.

```
1. Tables without references (Team, Categories, etc.)
2. Tables with simple Refs (Projects → Team)
3. Tables with multiple Refs (Tasks → Projects, Team, Tasks)
```

---

## Column Properties

### Available Properties

| Property | Type | Description |
|-----------|------|-------------|
| `id` | string | Technical identifier of the column |
| `type` | string | Data type (see types below) |
| `label` | string | Display name in the Grist interface |
| `widgetOptions` | string (JSON) | Widget options (choices, alignment, etc.) |
| `visibleCol` | number | ID of the column to display for Refs |
| `formula` | string | Python formula for calculated columns |
| `isFormula` | boolean | true if calculated column |

### Column Types

| Type | Description | Special Options |
|------|-------------|-------------------|
| `Text` | Simple text | — |
| `Int` | Integer | — |
| `Numeric` | Decimal | — |
| `Bool` | Boolean | — |
| `Date` | Date (Unix timestamp) | — |
| `DateTime` | Date and time | — |
| `Choice` | Single choice | `widgetOptions.choices` |
| `ChoiceList` | Multiple choices | `widgetOptions.choices` |
| `Ref:Table` | Reference | `visibleCol` |
| `RefList:Table` | List of references | `visibleCol` |
| `Attachments` | Files | — |

---

## widgetOptions

### Format
```javascript
widgetOptions: JSON.stringify({
    choices: ['option1', 'option2', 'option3'],
    choiceOptions: {
        'option1': { fillColor: '#ef4444', textColor: '#ffffff' },
        'option2': { fillColor: '#10b981', textColor: '#ffffff' }
    },
    alignment: 'center'  // 'left', 'center', 'right'
})
```

### Examples

#### Choice with colors
```javascript
{
    id: 'statut',
    type: 'Choice',
    label: 'Statut',
    widgetOptions: JSON.stringify({
        choices: ['todo', 'inprogress', 'review', 'done'],
        choiceOptions: {
            'todo': { fillColor: '#6b7280', textColor: '#ffffff' },
            'inprogress': { fillColor: '#3b82f6', textColor: '#ffffff' },
            'review': { fillColor: '#f59e0b', textColor: '#ffffff' },
            'done': { fillColor: '#10b981', textColor: '#ffffff' }
        }
    })
}
```

#### Simple Choice
```javascript
{
    id: 'priorite',
    type: 'Choice',
    label: 'Priorité',
    widgetOptions: JSON.stringify({
        choices: ['1 - Critique', '2 - Haute', '3 - Moyenne', '4 - Basse']
    })
}
```

#### ChoiceList (tags)
```javascript
{
    id: 'tags',
    type: 'ChoiceList',
    label: 'Tags',
    widgetOptions: JSON.stringify({
        choices: ['urgent', 'bug', 'feature', 'documentation', 'test']
    })
}
```

---

## References (Ref / RefList)

### visibleCol

`visibleCol` is the **numeric ID** of the column to display (not its name).

**Problem**: The ID is not known before creation.

**Solutions**:

#### 1. Create without visibleCol, then modify
```javascript
// Step 1: Create the Ref column without visibleCol
await grist.docApi.applyUserActions([
    ['AddColumn', 'Tasks', 'projet', { type: 'Ref:Projects', label: 'Projet' }]
]);

// Step 2: Get the ID of the 'nom' column in Projects
const colInfo = await getColumnId('Projects', 'nom');

// Step 3: Modify the column to add visibleCol
await grist.docApi.applyUserActions([
    ['ModifyColumn', 'Tasks', 'projet', { visibleCol: colInfo.id }]
]);
```

#### 2. Use the REST API to retrieve IDs
```javascript
async function getColumnId(tableId, colName) {
    // Via fetch if available
    const resp = await fetch(`/api/docs/${docId}/tables/${tableId}/columns`);
    const cols = await resp.json();
    return cols.columns.find(c => c.id === colName);
}
```

#### 3. Pragmatic pattern (without visibleCol)
```javascript
// Grist displays the ID by default if visibleCol is not set
// The user can configure manually in the interface
{
    id: 'projet',
    type: 'Ref:Projects',
    label: 'Projet'
}
```

---

## Complete Schema — TaskFlow Example

```javascript
const TASKFLOW_SCHEMA = {
    // ═══════════════════════════════════════════════════════
    // ORDER 1: Tables without external references
    // ═══════════════════════════════════════════════════════
    Team: {
        columns: [
            { id: 'nom', type: 'Text', label: 'Nom' },
            { id: 'email', type: 'Text', label: 'Email' },
            { id: 'avatar', type: 'Text', label: 'Avatar URL' },
            { id: 'role', type: 'Choice', label: 'Rôle',
              widgetOptions: JSON.stringify({
                  choices: ['Chef de projet', 'Développeur', 'Designer', 'Data analyst', 'Admin']
              })
            },
            { id: 'actif', type: 'Bool', label: 'Actif' }
        ]
    },

    // ═══════════════════════════════════════════════════════
    // ORDER 2: Tables with simple Refs
    // ═══════════════════════════════════════════════════════
    Projects: {
        columns: [
            { id: 'nom', type: 'Text', label: 'Nom' },
            { id: 'couleur', type: 'Text', label: 'Couleur' },
            { id: 'dateDebut', type: 'Date', label: 'Date début' },
            { id: 'dateFin', type: 'Date', label: 'Date fin' },
            { id: 'responsable', type: 'Ref:Team', label: 'Responsable' },
            { id: 'actif', type: 'Bool', label: 'Actif' }
        ],
        // References to configure after creation
        refs: [
            { column: 'responsable', visibleColName: 'nom' }
        ]
    },

    // ═══════════════════════════════════════════════════════
    // ORDER 3: Tables with multiple Refs
    // ═══════════════════════════════════════════════════════
    Tasks: {
        columns: [
            { id: 'titre', type: 'Text', label: 'Titre' },
            { id: 'description', type: 'Text', label: 'Description' },
            { id: 'dateDebut', type: 'Date', label: 'Date début' },
            { id: 'dateEcheance', type: 'Date', label: 'Échéance' },
            { id: 'priorite', type: 'Choice', label: 'Priorité',
              widgetOptions: JSON.stringify({
                  choices: ['1 - Critique', '2 - Haute', '3 - Moyenne', '4 - Basse'],
                  choiceOptions: {
                      '1 - Critique': { fillColor: '#ef4444' },
                      '2 - Haute': { fillColor: '#f59e0b' },
                      '3 - Moyenne': { fillColor: '#3b82f6' },
                      '4 - Basse': { fillColor: '#6b7280' }
                  }
              })
            },
            { id: 'statut', type: 'Choice', label: 'Statut',
              widgetOptions: JSON.stringify({
                  choices: ['todo', 'inprogress', 'review', 'done'],
                  choiceOptions: {
                      'todo': { fillColor: '#6b7280', textColor: '#fff' },
                      'inprogress': { fillColor: '#3b82f6', textColor: '#fff' },
                      'review': { fillColor: '#f59e0b', textColor: '#fff' },
                      'done': { fillColor: '#10b981', textColor: '#fff' }
                  }
              })
            },
            { id: 'progression', type: 'Numeric', label: 'Progression %' },
            { id: 'projet', type: 'Ref:Projects', label: 'Projet' },
            { id: 'assignees', type: 'RefList:Team', label: 'Assignés' },
            { id: 'type', type: 'Choice', label: 'Type',
              widgetOptions: JSON.stringify({
                  choices: ['tache', 'jalon', 'reunion']
              })
            },
            { id: 'dependDe', type: 'RefList:Tasks', label: 'Dépend de' },
            { id: 'tags', type: 'ChoiceList', label: 'Tags',
              widgetOptions: JSON.stringify({
                  choices: ['urgent', 'bloquant', 'documentation', 'test', 'bug']
              })
            },
            { id: 'estimationH', type: 'Numeric', label: 'Estimation (h)' },
            { id: 'tempsPasse', type: 'Numeric', label: 'Temps passé (h)' },
            { id: 'couleur', type: 'Text', label: 'Couleur' }
        ],
        refs: [
            { column: 'projet', visibleColName: 'nom' },
            { column: 'assignees', visibleColName: 'nom' },
            { column: 'dependDe', visibleColName: 'titre' }
        ]
    }
};
```

---

## Complete ensureSchema Function

```javascript
async function ensureSchema(SCHEMA) {
    const log = (msg) => console.log(`[Schema] ${msg}`);
    const tableOrder = Object.keys(SCHEMA);
    const tablesCreated = [];

    // ═══════════════════════════════════════════════════════
    // PHASE 1: Create tables in order
    // ═══════════════════════════════════════════════════════
    for (const tableName of tableOrder) {
        const tableSpec = SCHEMA[tableName];
        const columns = tableSpec.columns || tableSpec; // Support legacy format

        try {
            // Check if table exists
            const existing = await grist.docApi.fetchTable(tableName);
            const existingCols = Object.keys(existing).filter(c => c !== 'id' && c !== 'manualSort');
            log(`✓ Table ${tableName} exists (${existingCols.length} columns)`);

            // Add missing columns
            for (const col of columns) {
                if (!existingCols.includes(col.id)) {
                    try {
                        const colDef = buildColumnDef(col);
                        await grist.docApi.applyUserActions([
                            ['AddColumn', tableName, col.id, colDef]
                        ]);
                        log(`  + ${tableName}.${col.id} added`);
                    } catch (e) {
                        log(`  ~ ${tableName}.${col.id} skipped: ${e.message}`);
                    }
                }
            }

        } catch (e) {
            // Table does not exist → create it
            try {
                const colSpecs = columns.map(buildColumnDef);
                await grist.docApi.applyUserActions([
                    ['AddTable', tableName, colSpecs]
                ]);
                log(`✅ Table ${tableName} created`);
                tablesCreated.push(tableName);
            } catch (e2) {
                log(`❌ Error creating ${tableName}: ${e2.message}`);
            }
        }
    }

    // ═══════════════════════════════════════════════════════
    // PHASE 2: Configure visibleCol for references
    // ═══════════════════════════════════════════════════════
    for (const tableName of tableOrder) {
        const tableSpec = SCHEMA[tableName];
        const refs = tableSpec.refs || [];

        for (const ref of refs) {
            try {
                await configureVisibleCol(tableName, ref.column, ref.visibleColName);
                log(`  → ${tableName}.${ref.column} displays ${ref.visibleColName}`);
            } catch (e) {
                log(`  ~ visibleCol ${tableName}.${ref.column} not configured: ${e.message}`);
            }
        }
    }

    // ═══════════════════════════════════════════════════════
    // PHASE 3: Seed data if new tables
    // ═══════════════════════════════════════════════════════
    if (tablesCreated.length > 0) {
        log(`Tables created: ${tablesCreated.join(', ')}`);
        // Call seedData() if needed
    }

    return { tablesCreated };
}

// ─────────────────────────────────────────────────────────
// Helpers
// ─────────────────────────────────────────────────────────

function buildColumnDef(col) {
    const def = {
        id: col.id,
        type: col.type
    };
    if (col.label) def.label = col.label;
    if (col.widgetOptions) def.widgetOptions = col.widgetOptions;
    if (col.formula) {
        def.formula = col.formula;
        def.isFormula = true;
    }
    return def;
}

async function configureVisibleCol(tableName, refColName, visibleColName) {
    // Retrieve the column type to extract the target table
    // Ex: 'Ref:Projects' → 'Projects'
    const tableData = await grist.docApi.fetchTable(tableName);

    // Note: This function requires access to the REST API to
    // retrieve the numeric ID of the column.
    // In practice, you can let Grist use the default column
    // or configure manually in the interface.

    // Alternative pattern with ModifyColumn if the ID is known:
    // await grist.docApi.applyUserActions([
    //     ['ModifyColumn', tableName, refColName, { visibleCol: colId }]
    // ]);
}
```

---

## ModifyColumn — Modifying an existing column

```javascript
// Change the type
await grist.docApi.applyUserActions([
    ['ModifyColumn', 'Tasks', 'priorite', { type: 'Int' }]
]);

// Change the label
await grist.docApi.applyUserActions([
    ['ModifyColumn', 'Tasks', 'titre', { label: 'Titre de la tâche' }]
]);

// Add choices to an existing Choice column
await grist.docApi.applyUserActions([
    ['ModifyColumn', 'Tasks', 'statut', {
        widgetOptions: JSON.stringify({
            choices: ['todo', 'inprogress', 'review', 'done', 'cancelled']
        })
    }]
]);

// Configure visibleCol (if the numeric ID is known)
await grist.docApi.applyUserActions([
    ['ModifyColumn', 'Tasks', 'projet', { visibleCol: 2 }]
]);
```

---

## Pattern: Creation with verification

```javascript
async function safeCreateTable(tableName, columns) {
    try {
        await grist.docApi.fetchTable(tableName);
        console.log(`Table ${tableName} already exists`);
        return false;
    } catch {
        await grist.docApi.applyUserActions([
            ['AddTable', tableName, columns.map(buildColumnDef)]
        ]);
        console.log(`Table ${tableName} created`);
        return true;
    }
}

async function safeAddColumn(tableName, col) {
    try {
        const data = await grist.docApi.fetchTable(tableName);
        if (col.id in data) {
            console.log(`Column ${tableName}.${col.id} already exists`);
            return false;
        }
    } catch {
        console.log(`Table ${tableName} does not exist`);
        return false;
    }

    await grist.docApi.applyUserActions([
        ['AddColumn', tableName, col.id, buildColumnDef(col)]
    ]);
    console.log(`Column ${tableName}.${col.id} added`);
    return true;
}
```

---

## Calculated columns (formulas)

```javascript
{
    id: 'fullName',
    type: 'Text',
    label: 'Nom complet',
    formula: '$prenom + " " + $nom',
    isFormula: true
}

{
    id: 'joursRestants',
    type: 'Int',
    label: 'Jours restants',
    formula: '($dateEcheance - TODAY()).days if $dateEcheance else None',
    isFormula: true
}

{
    id: 'estEnRetard',
    type: 'Bool',
    label: 'En retard',
    formula: '$dateEcheance and $dateEcheance < TODAY() and $statut != "done"',
    isFormula: true
}
```

---

## Schema creation checklist

```
□ Define table order (without refs → with refs)
□ Add label to each column
□ Configure widgetOptions.choices for Choice/ChoiceList
□ Add choiceOptions for colors if needed
□ Declare refs to configure (visibleColName)
□ Plan calculated columns (formula)
□ Implement seedData() for initial data
□ Test in full creation mode
□ Test in missing columns mode
```
