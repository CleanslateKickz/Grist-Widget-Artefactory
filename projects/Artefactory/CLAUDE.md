# Project: Artefactory AI v11

Standalone Grist widget (single `index.html` file, ~2200 lines) serving as a no-code/low-code
IDE embedded in a Grist document. Users can create, edit, and preview "artefacts" (React
components, HTML, Grist widgets, SVG, Mermaid, Markdown) stored directly as rows in a
Grist table.

## Context

- **Use case**: allow a Grist user to write and test custom widgets without leaving the
  document. Each artefact is a row in the `Artefacts` table.
- **No build**: everything runs in the browser via CDN (Ace Editor, Babel standalone,
  React 18, Tailwind CDN, Mermaid, html2canvas, marked).
- **UI in French only**.
- **No app runtime** in this version: the IDE cannot compose multiple artefacts into a
  multi-route application. It is a simple factory, not a runtime.

## Architecture

Single file: [index.html](index.html). Sections separated by `// ============ XXX ============` markers:

| Section | Lines (≈) | Role |
|---------|-----------|------|
| `CONFIGURATION` | 602 | `TABLE_NAME`, `TABLE_COLS`, `TYPES` (templates per type) |
| `STATE` | 743 | Global `state` object |
| `GRIST BRIDGE` | 782 | `GristBridgeParent` parent side + `GRIST_BRIDGE_SCRIPT` injected in iframe |
| `INITIALIZATION` | 941 | `init()`, `ensureTable`, `loadArtefacts`, `loadGristSchema` (lazy) |
| `DOCUMENTATION UI` | 1119 | Selectable Markdown doc chips as AI context |
| `UI UPDATES` | 1186 | `showInit`, `updateSelect`, `selectArtefact`, etc. |
| `CODE EDITOR EVENTS` | 1377 | Ace hot-reload |
| `TYPE DETECTION` | 1426 | `detectType` (fallback only, Grist `Type` overrides) |
| `RENDER` | 1456 | `render`, `renderReact`, `renderHTML`, `renderGristWidget`, `renderFrame`, `renderSVG`, `renderMermaid`, `renderMarkdown` |
| `SPLITTER` | 1605 | Drag to resize code/preview |
| `VIEW CONTROLS` | 1651 | `setView`, `setDevice`, `toggleConsole`, etc. |
| `CONSOLE` | 1721 | Capture of `console.log/error` from iframe |
| `AI ASSISTANT` | 1756 | Quick prompt, AI panel, `sendAIRequest`, `buildAIContext`, `handleAIResponse`, auto-correction |
| `KEYBOARD` | 2034 | Ctrl+S, Ctrl+K, Escape |
| `CREATE / SAVE` | 2052 | Creation modal, `applyUserActions` persistence |

### The AI Panel

The AI assistant is **not** an agent. It sends a JSON POST to a user-configured
webhook (`webhookUrl` + optional `apiKey`) and automatically applies the returned code.
All intelligence is on the webhook side.

**To connect an agent (Claude, GPT, n8n, custom agent): see [README.md](README.md).**

### Artefact Types

Defined in `TYPES` ([index.html:614](index.html#L614)):

| Type | Ace Mode | Rendering |
|------|----------|-----------|
| `react` | jsx | Babel standalone in-iframe, React 18, Recharts available, inlined UI components (Card, Button, Input, Badge) |
| `html` | html | iframe srcdoc with Tailwind CDN |
| `grist` | html | iframe + Grist bridge injected in place of `grist-plugin-api.js` |
| `svg` | svg | inline rendering (scripts stripped) |
| `mermaid` | text | `mermaid.render` loaded on demand |
| `markdown` | markdown | `marked.parse` |

### Grist Bridge

Classic pattern: Artefactory iframes cannot access the Grist API directly
(sandbox + cross-origin via srcdoc). The bridge:

1. Parent side: `GristBridgeParent` class that listens for `postMessage` from iframes
   and executes `grist.docApi.*` calls on their behalf.
2. Iframe side: `GRIST_BRIDGE_SCRIPT` (string literal) is **injected in place of**
   the `<script src="grist-plugin-api.js">` tag via `injectGristBridge()`.
   It exposes a fake `window.grist` that proxies via `postMessage`.
3. **Only one active iframe at a time**: `setCurrentIframe()` clears callbacks on
   each render. No nested iframe support (see difference noted below).

### Grist Schema (AI Option)

`loadGristSchema()` snapshots all non-`_grist` tables in the doc with 3 sample rows
per table. **Loaded lazily** since the May 2026 fix:
- Not loaded at `init()` (avoids N×fetchTable at startup)
- Loaded on first `change` of the "📊 Schéma Grist" checkbox
- Loaded as fallback in `buildAIContext()` if the checkbox is checked but
  `state.gristSchema` is null (case of a checkbox restored from localStorage)

## Project-Specific Conventions

- **No framework** in the widget itself (vanilla JS + Ace + marked).
- **Tailwind via CDN** in templates (never in the host widget).
- **All comments and user messages in French.**
- **No emojis in code** (except in user-visible status strings — that's the widget's
  visual identity).
- **localStorage persistence** under the key `artefactory_ai_config_v11`.
- **`grist.ready({ requiredAccess: 'full' })`**: required because the widget
  creates and modifies the `Artefacts` table.

### `Artefacts` Table Schema

Auto-created by `ensureTable()` on first launch.

| Column | Grist Type | Notes |
|--------|-----------|-------|
| `Nom` | Text | Displayed in the selector |
| `Type` | Choice | `react / html / grist / svg / mermaid / markdown` |
| `Code` | Text | Full source code |
| `Description` | Text | Optional |
| `Tags` | ChoiceList | **Declared but unused by the widget**. Kept for compatibility with users who use it in Grist directly. Do not read/write from widget code until a feature uses it. |
| `IsDoc` | Bool | If true + Type=markdown → artefact appears in the "Contextual Documentation" chips of the AI panel |
| `CreatedAt` | DateTime | Unix seconds (Grist convention) |

## Current Status

- **Bugs fixed (May 2026)**:
  - Infinite auto-correction loop (`aiCorrectionCount` reset each turn) → fixed via `isAutoCorrect` parameter
  - Potential XSS in `addAIMessage` (innerHTML on webhook response) → fixed via DOM construction (textContent + `<br>`)
  - `loadGristSchema` eagerly at `init()` → fixed lazy
  - Post-creation race (`state.artefacts[length-1]`) → fixed via `result.retValues[0]`
  - `detectType` false positives (markdown on backticks, svg on inline xmlns) → fixed via stricter patterns
  - `cleanReactCode` didn't handle multi-line imports → fixed via `[\s\S]*?`
- **Not yet implemented**: partial AI operations (replace lines / insert at line). The widget applies either the entire buffer or the selection.
- **Not yet implemented**: AI response streaming. It's a one-shot POST.

## Points of Attention

### Iframe sandbox `allow-scripts allow-same-origin`

Combination that cancels origin isolation (since srcdoc is necessarily same-origin).
Required for:
- `iframe.contentDocument` read by `html2canvas` during screenshot capture
- `console` patching (the `window.parent.postMessage` hook from the iframe)

**Consequence**: a malicious artefact can access the host widget's DOM. To keep in mind
if secrets are added to the widget. The n8n API key stored in localStorage **is
readable** by any rendered artefact — acceptable because it's the user's own key,
but should be documented for future multi-tenancy.

### Truncated Conversation History

`buildAIContext` sends the last 10 messages with `content.substring(0, 500)`
([index.html:1968](index.html#L1968)). If the conversation contains a long code block
in a message, it is silently truncated. Don't expect perfect conversational context.

### Hot-reload Only in `split` Mode

`onCodeChange` only triggers re-render if `state.view === 'split'`
([index.html:1397](index.html#L1397)). In `code`-only mode, use Ctrl+Enter or switch
views. Deliberate design choice.

### No CSRF Protection on Webhook

The AI webhook is called from the user's browser with their token. If you configure
a public webhook without Bearer token, anyone can hit it. Always configure auth on
the webhook side (Bearer or IP allowlist).

### Auto-correction: 3 turns max per user send

Counter `aiCorrectionCount` resets only when the user sends a new prompt (not during
an auto-correction). After 3 consecutive corrections, the widget gives up and
displays "Limit reached (3/3)".

### Difference from the Old Artefactory v12 (widget_app/)

This v11 is intentionally simpler than the historical v12
(`projects/widget_app/app.html`):
- No Edit/Run runtime with sidebar and route navigation
- No `app` (manifest) or `component` (reusable React component) types
- No component inclusion system (`window.app.include()`)
- Adds however: Mermaid, contextual documentation (`IsDoc`)

If multi-artefact composition needs to be re-introduced, start from the patterns
documented in [../widget_app/CLAUDE.md](../widget_app/CLAUDE.md) (notably
`prepareNestedHtml` and the `window.parent.parent` retarget).

## Files

- [index.html](index.html) — complete widget
- [README.md](README.md) — installation + **connecting an AI agent** (HTTP contract, widget limits, agent patterns)
- [AGENT_SYSTEM_PROMPT.md](AGENT_SYSTEM_PROMPT.md) — ready-to-use system prompt for the agent (copy-paste on webhook side)
