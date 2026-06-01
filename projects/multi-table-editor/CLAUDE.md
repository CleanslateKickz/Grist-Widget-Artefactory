# Project: Multi-Table Editor

## Context
Single-interface CRUD widget for browsing and editing all tables in a Grist document. In a commercial real estate CRM, this is used to quickly navigate between Owners, Properties, Tenants, Leases, and any other custom tables without switching Grist views.

## Architecture
- Single standalone file: `index.html` (~1375 lines)
- Tab navigation generated dynamically from `_grist_Tables` / `_grist_Tables_column` schema
- Per-table CRUD: Add, Edit, Delete with modal forms
- Reference field support (Ref:, RefList:) with auto-loaded dropdown options
- Read-only detection (formula-only columns are excluded from editing)

## Specific conventions
- Schema loaded from Grist internal tables (`_grist_Tables`, `_grist_Tables_column`)
- Internal tables and `GristDocTutorial` tables are filtered out
- Formula columns detected via `isFormula` flag and shown as read-only
- Reference fields try multiple display field names (name, nom, title, titre, label, description)
- Input type inferred from field name suffixes (email, url, comment, description, adresse)
- ManualSortPos and gristHelper columns are hidden

## Current state
- Complete CRUD working with all field types
- Reference field loading works for single and multi-select
- Modal editing with date/datetime handling (Unix timestamps)
- Record deletion with confirmation dialog
- Schema refresh button to reload table structure

## Points of attention
- `grist.docApi.fetchTable` called on internal Grist tables — these may change with Grist versions
- Reference data loaded per-field on modal open — may be slow with many records
- Demo mode not implemented (requires Grist API to function)
- No pagination — large tables render all rows in a single HTML table
- Uses `applyUserActions` directly — no undo batching
