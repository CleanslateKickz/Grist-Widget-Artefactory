# Skills - Grist Development Patterns

This folder contains **reusable code patterns** for Grist widget development.

## For AI Agents

**Before coding a Grist widget, consult these files to use the standard patterns.**

## Skills Index

| File | Content |
|---------|---------|
| [schema.md](schema.md) | ⭐ **Complete schema creation**: tables, columns, labels, refs, choices |
| [grist-api.md](grist-api.md) | Grist API CRUD: ready, fetchTable, applyUserActions |
| [data-conversion.md](data-conversion.md) | Columnar → objects conversion, dates, RefList |
| [inter-widget.md](inter-widget.md) | Inter-widget communication: selection, options, events |
| [bridge.md](bridge.md) | GristBridge for sandboxed iframes |
| [patterns.md](patterns.md) | UI patterns: modals, filters, toasts |

## Quick Usage

### Minimal Widget
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <script src="https://docs.getgrist.com/grist-plugin-api.js"></script>
</head>
<body>
    <div id="app">Loading...</div>
    <script>
        // See skills/grist-api.md for full patterns
        grist.ready({ requiredAccess: 'read table' });
        grist.onRecords(records => {
            // records is in columnar format, see data-conversion.md
        });
    </script>
</body>
</html>
```

## Sources

Patterns extracted from:
- `projects/tasks_app/` — TaskFlow (Kanban, Gantt, Calendar)
- `projects/widget_app/` — Artefactory (No-code IDE)
- [nic01asfr/grist-widgets](https://github.com/nic01asfr/grist-widgets) — Geo-map, Cluster Quest
- [nic01asfr/grist-navigation-widgets](https://github.com/nic01asfr/grist-navigation-widgets) — Multi-widget navigation
- [nic01asfr/Grist-App-Nest](https://github.com/nic01asfr/Grist-App-Nest) — Dynamic dashboard
