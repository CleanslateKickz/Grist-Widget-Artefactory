# Multi-Table Editor

> Single-interface CRUD widget for all tables in your Grist document. Built for commercial real estate CRM workflows.

## Features
- **Tab navigation** — switch between all user tables (Owners, Properties, Tenants, Leases, etc.)
- **Full CRUD** — Add, Edit, Delete records from any table
- **Reference fields** — Auto-loaded dropdowns for `Ref:` and `RefList:` columns (e.g., link an owner to a property)
- **Smart field detection** — Auto-detects field types: text, number, date, boolean, email, URL, textarea
- **Read-only awareness** — Formula-only tables show as read-only
- **Schema refresh** — Reload table structure on demand

## Installation
1. In Grist: **Add Widget → Custom → Enter Custom URL**
2. URL: `https://CleanslateKickz.github.io/Grist-Widget-Artefactory/multi-table-editor/`
3. Grant **"Full document access"**

## Usage
- Tables appear as tabs at the top
- Click a tab to browse that table's records
- Use **Add record** to create a new entry
- Click the edit/pencil icon to modify a record
- Click the trash icon to delete
- Reference fields (linked records) show a dropdown of available options

## CRE CRM Use Cases
- **Quick data entry** — Add a new tenant or owner without navigating away
- **Cross-table editing** — Fix a lease date, then switch to update the property address
- **Reference management** — Link properties to owners via the Ref: dropdown

## Schema
The widget auto-detects all non-internal tables. It reads the `_grist_Tables` and `_grist_Tables_column` system tables to build the interface — no manual configuration needed.
