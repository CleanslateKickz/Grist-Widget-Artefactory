# CLAUDE.md

Development guide for the Grist widgets repository.

---

## Instructions for AI agents

**This repo follows a strict development / production separation.**

### Essential rules

1. **Never modify `published/`** unless explicitly asked to publish
2. **Develop in `projects/`** — all projects are in this folder
3. **Each project has its own CLAUDE.md** — read it before any intervention
4. **The manifest.json is auto-generated** — never edit it manually
5. **Consult `skills/`** — reusable code patterns for Grist

### Checklist before coding

```
□ Read the CLAUDE.md of the relevant project
□ Consult skills/ for standard patterns
□ Understand the existing architecture
□ Identify functions/patterns already present
□ Do not duplicate what already exists
```

### Workflow

```
DEVELOPMENT                           PUBLICATION
────────────────────────────────────────────────────────
projects/mon-widget/     ──promote──►  published/mon-widget/
    ├── files.html                        ├── package.json (required)
    └── CLAUDE.md                          └── index.html
```

### Before coding on a project

1. Read the project's `CLAUDE.md` (e.g.: `projects/tasks_app/CLAUDE.md`)
2. Understand the existing architecture
3. Do not publish without an explicit request

### When the user asks to "publish"

1. Create `published/widget-name/package.json` with the `grist` section
2. Copy the final files to `published/widget-name/`
3. Run `npm run manifest` to regenerate the catalog
4. Commit with a descriptive message

### Code conventions

- **Static widgets** : standalone HTML with `<script src="grist-plugin-api.js">`
- **English** for comments and user messages
- **No frameworks** unless the project specifies it
- **grist.ready()** required with appropriate `requiredAccess`

---

## Overview

This repository contains custom widgets for [Grist](https://www.getgrist.com/). It is structured to support:
- **Development** of widgets (work area)
- **Publication** of stable widgets (area deployed on GitHub Pages)
- Both types of Grist widgets: **static** (pure HTML) and **build** (npm/React)

## Repository structure

```
Widgets-Grist/
├── .github/
│   └── workflows/
│       └── publish.yml           # CI/CD : build + deploy on GitHub Pages
│
├── .nojekyll                     # Disables Jekyll on GitHub Pages
├── .gitignore
├── package.json                  # npm workspaces config
├── CLAUDE.md                     # This file
├── README.md                     # Public documentation
│
├── projects/                     # DEVELOPMENT AREA
│   ├── tasks_app/               # TaskFlow (kanban, gantt, calendar)
│   │   ├── CLAUDE.md
│   │   ├── kanban.html
│   │   ├── gantt.html
│   │   ├── calendar.html
│   │   └── ...
│   │
│   └── widget_app/              # Artefactory (no-code IDE)
│       ├── CLAUDE.md
│       ├── app.html
│       ├── app_runtime.html
│       └── templates/
│
├── published/                    # PUBLISHED AREA (deployed on gh-pages)
│   ├── manifest.json            # Widget catalog (auto-generated)
│   │
│   ├── taskflow/                # Published TaskFlow widgets
│   │   ├── package.json
│   │   ├── kanban/
│   │   │   └── index.html
│   │   ├── gantt/
│   │   │   └── index.html
│   │   └── calendar/
│   │       └── index.html
│   │
│   ├── artefactory/             # Published Artefactory widgets
│   │   ├── package.json
│   │   ├── admin/
│   │   │   └── index.html
│   │   ├── runtime/
│   │   │   └── index.html
│   │   ├── registry.json
│   │   └── components/
│   │       └── ...
│   │
│   └── [other-widgets]/
│
├── packages/                     # WIDGETS WITH BUILD (optional)
│   └── [widget-react]/
│       ├── package.json
│       ├── src/
│       └── dist/                # Output → copied to published/
│
├── skills/                       # REUSABLE CODE PATTERNS
│   ├── README.md                # Skills index
│   ├── schema.md                # ⭐ Schema creation (tables, columns, refs, labels)
│   ├── grist-api.md             # Grist CRUD API
│   ├── data-conversion.md       # Columnar conversion, dates, RefList
│   ├── inter-widget.md          # Inter-widget communication
│   ├── bridge.md                # GristBridge for iframes
│   └── patterns.md              # Modals, filters, UI patterns
│
└── scripts/
    ├── generate-manifest.js     # Generates manifest.json from published/
    └── promote.js               # Copies from projects/ to published/
```

## Repository areas

### `projects/` — Development

Work area for widgets under development. **Not deployed** on GitHub Pages.

- Each project has its own `CLAUDE.md` with specifics
- Files can be tested locally (demo mode) or via raw GitHub URL
- No strict structure constraints

### `published/` — Production

Area for published stable widgets. **Deployed on GitHub Pages** via CI/CD.

- Each widget has a `package.json` with the `grist` section (metadata)
- Required structure: `widget-name/index.html` (or `widget-name.html`)
- The `manifest.json` is auto-generated by the script

### `packages/` — Widgets with build

For widgets requiring compilation (React, Vue, TypeScript...).

- Each package has its own `package.json` with build scripts
- The build output goes to `published/` via the build script
- Uses npm workspaces for dependency management

## Development workflow

### 1. Develop a widget

```bash
# Work in projects/
cd projects/mon-widget/

# Test locally (open in browser = demo mode)
# Or test with Grist via raw GitHub URL
```

### 2. Promote to published/

```bash
# When the widget is ready
npm run promote -- mon-widget

# Or manually: copy files to published/
```

### 3. Publish

```bash
# Generate the manifest
npm run manifest

# Commit and push to main
git add .
git commit -m "Publish mon-widget v1.0"
git push

# GitHub Actions deploys automatically to gh-pages
```

## Grist configuration

### Published widget URLs

```
https://[USER].github.io/Widgets-Grist/taskflow/kanban/
https://[USER].github.io/Widgets-Grist/artefactory/runtime/
```

### Configure as widget source

For a self-hosted Grist instance, set the environment variable:

```bash
GRIST_WIDGET_LIST_URL=https://[USER].github.io/Widgets-Grist/manifest.json
```

The widgets will appear in the Grist "Custom Widget" selector.

## Widget structure

### Static widget (no build)

```
published/mon-widget/
├── package.json      # Required metadata
├── index.html        # Entry point
├── style.css         # Optional
└── script.js         # Optional
```

**Minimal package.json:**

```json
{
  "name": "@org/widget-mon-widget",
  "version": "1.0.0",
  "grist": {
    "widgetId": "@org/widget-mon-widget",
    "name": "Mon Widget",
    "url": "https://[USER].github.io/Widgets-Grist/mon-widget/",
    "accessLevel": "full",
    "description": "Widget description"
  }
}
```

### Widget with build (npm/React)

```
packages/mon-widget-react/
├── package.json
├── src/
│   ├── App.tsx
│   └── index.tsx
├── vite.config.ts
└── dist/             # → copied to published/mon-widget-react/
```

**package.json:**

```json
{
  "name": "@org/widget-mon-widget-react",
  "version": "1.0.0",
  "scripts": {
    "dev": "vite",
    "build": "vite build --outDir ../../published/mon-widget-react"
  },
  "grist": {
    "widgetId": "@org/widget-mon-widget-react",
    "name": "Mon Widget React",
    "url": "https://[USER].github.io/Widgets-Grist/mon-widget-react/",
    "accessLevel": "full"
  }
}
```

## `grist` fields in package.json

| Field | Required | Description |
|-------|----------|-------------|
| `widgetId` | Yes | Unique identifier (format: `@org/widget-name`) |
| `name` | Yes | Display name in the Grist selector |
| `url` | Yes | Full URL to the widget |
| `accessLevel` | No | `"none"`, `"read table"`, or `"full"` |
| `description` | No | Short description (~125 characters) |
| `renderAfterReady` | No | Rendering optimization (default: true) |
| `authors` | No | `[{ "name": "...", "url": "..." }]` |

## npm commands

```bash
# Install dependencies (workspaces)
npm install

# Generate manifest.json
npm run manifest

# Promote a widget from projects/ to published/
npm run promote -- project-name

# Build all widgets (packages/)
npm run build

# All in one: build + manifest
npm run deploy
```

## CI/CD with GitHub Actions

The `.github/workflows/publish.yml` workflow:

1. Triggers on push to `main` (if `published/` or `packages/` are modified)
2. Installs dependencies
3. Builds the widgets (packages/)
4. Generates the manifest.json
5. Deploys `published/` to the `gh-pages` branch

### GitHub Pages configuration

1. Settings → Pages
2. Source: Deploy from a branch
3. Branch: `gh-pages` / `/ (root)`

## Conventions

### Naming

- **Dev projects**: `project-name/` or `project-name-vX/` (with version)
- **Published widgets**: `widget-name/` (no version, version is in package.json)
- **Widget IDs**: `@org/widget-name`

### Versioning

- Use semver in `package.json`
- Git tag for releases: `v1.0.0`
- The manifest includes `lastUpdatedAt` automatically

### Commits

```
feat(taskflow): add drag-drop to kanban
fix(artefactory): bridge timeout issue
docs: update CLAUDE.md
chore: bump versions
```

## Current projects

### TaskFlow (`projects/tasks_app/`)

Suite of 3 Grist widgets for project management:
- **Kanban**: Board view with drag & drop
- **Gantt**: Gantt chart with dependencies
- **Calendar**: Monthly/weekly calendar view

See `projects/tasks_app/CLAUDE.md` for details.

### Artefactory (`projects/widget_app/`)

No-code IDE for composing Grist applications:
- **Admin**: Manifest configurator (to be created)
- **Runtime**: Application executor

See `projects/widget_app/CLAUDE.md` for details.

## Publishing guide

### Step by step

#### 1. Prepare the widget

The widget must be functional and tested. Check:
- [ ] Demo mode working (local opening)
- [ ] Grist integration working
- [ ] No console errors
- [ ] Responsive / usable

#### 2. Create the structure in published/

```bash
# Create the folder
mkdir -p published/mon-widget

# Create the package.json (required)
```

**published/mon-widget/package.json:**
```json
{
  "name": "mon-widget",
  "version": "1.0.0",
  "description": "Short widget description",
  "grist": {
    "widgetId": "mon-widget",
    "name": "Mon Widget",
    "accessLevel": "full",
    "description": "Description displayed in Grist"
  }
}
```

#### 3. Copy the files

```bash
# Copy the main HTML
cp projects/mon-widget/widget.html published/mon-widget/index.html

# Or use the script
npm run promote -- projects/mon-widget/widget.html mon-widget
```

#### 4. Generate the manifest

```bash
npm run manifest
```

Verifies that the widget appears in `published/manifest.json`.

#### 5. Commit and push

```bash
git add published/
git commit -m "feat: publish mon-widget v1.0.0"
git push
```

The GitHub Actions workflow deploys automatically.

### Multiple widgets in a package

A single `package.json` can declare multiple widgets:

```json
{
  "name": "taskflow",
  "grist": [
    {
      "widgetId": "taskflow-kanban",
      "name": "TaskFlow Kanban",
      "url": "kanban/index.html",
      "accessLevel": "full"
    },
    {
      "widgetId": "taskflow-gantt",
      "name": "TaskFlow Gantt",
      "url": "gantt/index.html",
      "accessLevel": "full"
    }
  ]
}
```

### Updating a widget

1. Modify the files in `published/`
2. Increment the version in `package.json`
3. `npm run manifest` + commit + push

---

## Per-project CLAUDE.md structure

Each project in `projects/` must have its own `CLAUDE.md` that documents:

```markdown
# Project: Project Name

## Context
[Objective and use case]

## Architecture
[File structure and their roles]

## Specific conventions
[Project-specific rules]

## Current state
[What works, what remains to be done]

## Points of attention
[Pitfalls, known bugs, technical decisions]
```

This allows AI agents to quickly understand the context without having to explore all the code.

---

## Skills — Code patterns

The `skills/` folder contains standard code patterns for Grist development.

### Usage

**Before coding**, consult the appropriate file:

| Need | File |
|------|------|
| **Create tables/columns/refs** | `skills/schema.md` ⭐ |
| Read/write Grist data | `skills/grist-api.md` |
| Convert columnar data | `skills/data-conversion.md` |
| Synchronize multiple widgets | `skills/inter-widget.md` |
| Widget with sub-iframes | `skills/bridge.md` |
| Modals, filters, toasts | `skills/patterns.md` |

### Principles

1. **Reuse** existing patterns rather than reinventing
2. **Consistency** — same code style throughout the repo
3. **Document** new patterns in skills/

---

## Multi-agent / multi-session collaboration

This repo is designed to support the work of **multiple AI agents** and **multiple users** in a consistent manner.

### Documentation architecture

```
CLAUDE.md (root)           ← Global rules, structure, workflow
    │
    ├── projects/*/CLAUDE.md ← Project-specific rules
    │
    └── skills/*.md          ← Reusable code patterns
```

### Protocol for a new agent/session

1. **Read this file** (root CLAUDE.md) first
2. **Identify the project** concerned by the request
3. **Read the project's CLAUDE.md** before any intervention
4. **Consult skills/** for the patterns to use
5. **Do not modify** what works without an explicit reason

### Consistency rules

#### Code
- Use the **same function names** as in existing projects
- Respect the project's **naming conventions**
- Keep the **code style** consistent (indentation, comments, etc.)
- **English** for user messages and comments

#### Mandatory patterns for Grist widgets
```javascript
// Standard initialization
grist.ready({ requiredAccess: 'full' | 'read table' | 'none' });

// Columnar → object conversion (see skills/data-conversion.md)
function convertToRows(data) { ... }

// Demo mode (fallback without Grist)
try { grist.ready(...); } catch { isDemo = true; loadDemoData(); }

// Inter-widget selection (see skills/inter-widget.md)
grist.setSelectedRows([id]);
grist.onRecord((record) => { ... });
```

#### Standard HTML structure
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <script src="https://docs.getgrist.com/grist-plugin-api.js"></script>
    <style>/* CSS variables, styles */</style>
</head>
<body>
    <div id="app">Loading...</div>
    <script>/* Code organized in sections */</script>
</body>
</html>
```

### Inter-session communication

Sessions do not communicate directly. Consistency is ensured by:

1. **Documentation** — everything is documented in the CLAUDE.md files
2. **Patterns** — use the patterns from skills/
3. **Git** — commits document the changes
4. **Current state** — each project's CLAUDE.md indicates the state

### Documentation updates

When an agent makes a significant change:

1. **Update the project's CLAUDE.md** if the architecture changes
2. **Add a pattern to skills/** if a new reusable pattern is created
3. **Document in the commit** what was done and why

### Anti-patterns to avoid

| Don't do this | Do this instead |
|---------------|-----------------|
| Modify `published/` without being asked | Work in `projects/` |
| Create a new pattern without checking skills/ | Reuse existing patterns |
| Change the architecture without documenting | Update the CLAUDE.md |
| Ignore the project's CLAUDE.md | Read it first |
| Guess the context | Read the existing code and docs |
| Duplicate code | Factorize or reference |

---

## Reference repositories

Additional patterns available in nic01asfr's GitHub repos:

| Repo | Useful content |
|------|---------------|
| [grist-widgets](https://github.com/nic01asfr/grist-widgets) | Geo-map, Cluster Quest, React build |
| [grist-navigation-widgets](https://github.com/nic01asfr/grist-navigation-widgets) | Multi-widget navigation, setCursorPos |
| [Grist-App-Nest](https://github.com/nic01asfr/Grist-App-Nest) | Dynamic dashboard, React in Grist |
| [mcp-server-grist](https://github.com/nic01asfr/mcp-server-grist) | MCP Server for Grist |

---

## Resources

- [Grist Custom Widgets Documentation](https://support.getgrist.com/widget-custom/)
- [Grist Plugin API](https://support.getgrist.com/code/modules/grist_plugin_api/)
- [gristlabs/grist-widget](https://github.com/gristlabs/grist-widget) (official repo)
- [GitHub Pages Documentation](https://docs.github.com/en/pages)
