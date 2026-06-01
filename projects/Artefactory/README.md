# Artefactory AI

No-code IDE embedded in Grist as a custom widget. Lets you create, edit,
and preview React components, HTML, Grist widgets, SVG, Mermaid, and
Markdown — directly as rows in a Grist table. Includes an AI assistant
that delegates all generation to a webhook you configure (Claude, GPT,
n8n, custom agent...).

> For development conventions and internal widget architecture,
> see [CLAUDE.md](CLAUDE.md).

## Installation in Grist

1. Open a Grist document (self-hosted or cloud).
2. Add a Custom Widget pointing to `index.html` (raw GitHub or GitHub Pages).
3. Set the access level to **Full** (the widget creates the `Artefacts` table on first launch).
4. On first launch: the `Artefacts` table appears automatically, ready to receive your first artefacts.

## Usage

| Action | Shortcut |
|--------|----------|
| Save current artefact to Grist | **Ctrl+S** |
| Open quick AI prompt | **Ctrl+K** |
| Execute current code | **Ctrl+Enter** |
| Exit fullscreen | **Escape** |

Available views: `Render` / `Code` / `Split` (default). Automatic hot-reload in `Split` mode (~800ms after last keystroke).

Three simulated devices: Desktop / Tablet / Mobile (frame around the iframe, purely visual — layout remains the responsibility of the artefact code).

## Contextual Documentation for AI

When you create a **Markdown** artefact and check **"This artefact is documentation"**:

- It appears as a clickable chip in the AI panel and in the quick prompt.
- When selected, its content is attached to each request sent to the webhook (`documentation` key).

This is the intended mechanism for passing to the agent: in-house code conventions, third-party table schemas, business rules, expected output format, etc.

## Connecting an AI Agent

The widget **does not embed any agent logic**. The assistant is a simple HTTP client that:

1. Builds a JSON body describing the state (current code, selection, conversation, docs, optional Grist schema, optional screenshot).
2. POSTs to the URL configured in the AI panel.
3. Automatically applies the returned code in the editor and re-triggers rendering.

**All intelligence is on the webhook side.** It is responsible for calling Claude / GPT / a custom agent and formatting the response.

### HTTP Contract

#### Request

```
POST <webhookUrl>
Content-Type: application/json
Authorization: Bearer <apiKey>     ← optional, only present if configured
```

Sent body (always-present keys in bold):

```jsonc
{
  "prompt":        "string",            // User-entered text
  "code":          "string",            // Full code OR selection (depending on codeContext)
  "codeContext":   "full" | "selection",
  "mode":          "react" | "html" | "grist" | "svg" | "mermaid" | "markdown",
  "artefactName":  "string",
  "artefactId":    123,
  "console":       ["msg1", "..."],     // Last 5 captured console errors
  "timestamp":     "2026-05-09T13:42:00.000Z",
  "useVision":     false,

  // Present ONLY if codeContext === "selection":
  "selection": { "startLine": 12, "endLine": 18, "startCol": 0, "endCol": 0 },
  "fullCode":  "full code (useful for context around the selection)",

  // Present ONLY if the user checked docs:
  "documentation": [
    {
      "id":          7,
      "name":        "In-house React Conventions",
      "description": "Naming rules, JSX structure...",
      "content":     "# Conventions\n\n- Always..."
    }
  ],

  // Present ONLY if the "Grist Schema" checkbox is checked:
  "gristSchema": {
    "Tasks": {
      "name":       "Tasks",
      "columns":    ["Title", "Status", "DueDate"],
      "sampleData": [{ "id": 1, "Title": "...", "Status": "open", "DueDate": 1715000000 }]
    }
    // ... one entry per non-_grist table
  },

  // Present ONLY if the 📸 button is active:
  "screenshot": "data:image/jpeg;base64,...",

  // Last 10 conversation messages, content truncated to 500 chars each:
  "conversation": [
    { "role": "user",      "content": "..." },
    { "role": "assistant", "content": "..." }
  ]
}
```

#### Expected Response

```jsonc
{
  "code":             "new code to apply",        // optional
  "message":          "Text displayed in chat",    // optional
  "replaceSelection": true,                         // optional, default true
  "error":            "error message"               // optional, displayed in red
}
```

Accepted aliases (see `handleAIResponse` in [index.html](index.html)):
- `code` can also be called `content` or `response`
- `message` can also be called `explanation`

#### Behaviors to Know

1. **Automatic application**: if the response contains `code`, it is immediately
   injected into Ace, saved in `state.code`, and re-rendering is triggered.
   **The agent does not need to ask for confirmation.**
2. **Selection vs full**: when `codeContext === "selection"`, by default the returned
   `code` **replaces only the selection**. To replace the entire buffer despite active
   selection: return `replaceSelection: false`.
3. **Auto-correction**: if a console error appears within ~1.5s of code application,
   the widget automatically sends a new request with `prompt: "Fix the following errors:\n…"`
   (up to 3 times). **The agent must therefore be idempotent** and able to fix code
   it produced itself.
4. **No streaming**: one-shot JSON POST. If you want streaming, you'll need to extend
   the widget's `sendAIRequest`.
5. **Default timeout**: 30 seconds on the browser `fetch` side (then standard network error).

### Widget Limits Your Agent Must Know

For the agent to generate code that **actually works** in Artefactory, it must respect the constraints below. An agent that invents an unlisted API will produce code that silently crashes on execution.

#### Available Grist APIs in the Bridge (closed list)

The widget injects a fake `window.grist` in the iframe ([index.html:904](index.html#L904))
that proxies to the real Grist. **Only** the following APIs are available:

| API | Available | Note |
|-----|-----------|------|
| `grist.ready(options)` | ✅ | Always call first |
| `grist.docApi.listTables()` | ✅ | Returns `string[]` |
| `grist.docApi.fetchTable(id)` | ✅ | Columnar format `{ id: [], Col: [] }` |
| `grist.docApi.applyUserActions(actions)` | ✅ | `AddRecord`, `UpdateRecord`, `RemoveRecord`, multi-actions |
| `grist.docApi.getAccessToken(opts)` | ✅ | For signed REST calls |
| `grist.onRecords(cb)` | ✅ | `(records, mappings) => void` |
| `grist.onRecord(cb)` | ✅ | `(record, mappings) => void`; `record` can be `null` |
| `grist.onOptions(cb)` | ✅ | Read widget options |
| `grist.setCursorPos({rowId})` | ✅ | Navigate to a row |
| `grist.getTable(id).{create,update}()` | ✅ | Syntactic sugar over `applyUserActions` |
| `grist.mapColumnNames(record, mappings)` | ✅ | Mapping helper |
| `grist.setOptions(opts)` | ❌ | **NOT exposed** by the bridge |
| `grist.selectedTable` | ❌ | **NOT exposed** — pass `tableId` via `mappings.tableId` or hardcode it |
| `window.app.*` (navigate / emit / on / state) | ❌ | **DOES NOT EXIST in v11** — that's a v12 feature (`projects/widget_app/`) |

If the agent generates `grist.setOptions(...)` or `grist.selectedTable`, the artefact will crash with `TypeError`. Same for `window.app.navigate(...)`.

#### Execution Environment per Artefact Type

**Type `react`** ([index.html:1475](index.html#L1475)):
- React 18 + ReactDOM 18 (UMD)
- Babel standalone (no module resolver → **no imports / exports**)
- Tailwind via CDN
- Recharts **2.12.7** (version it — if the agent suggests Recharts ≥ v3 code, it won't work)
- Global hooks: `useState, useEffect, useRef, useMemo, useCallback, useContext, useReducer, createContext, Fragment`
- Available inlined UI components:
  - `Card, CardHeader, CardTitle, CardContent`
  - `Button` with `variant="default|outline|ghost"` + `size="sm|default|lg"`
  - `Input`
  - `Badge` with `variant="default|success|destructive"` (**not** `outline`, do not invent it)
- Main component **must be named `App`** (or `Component` tolerated)
- ErrorBoundary wraps rendering — render errors are surfaced to the host widget

**Type `html`**:
- iframe srcdoc with automatic Tailwind CDN wrapper if no `<!DOCTYPE>`
- Vanilla JS ES6+, **no framework** except via explicit CDN
- Console captured and surfaced to widget

**Type `grist`**:
- Complete HTML document **required** (DOCTYPE + html/head/body)
- `<script src="https://docs.getgrist.com/grist-plugin-api.js">` is **automatically replaced** by the bridge
- All Grist APIs go through `postMessage` — asynchronous, ~1-5ms latency
- `grist.ready({ requiredAccess })` required (`'none' | 'read table' | 'full'`)

**Type `svg`**:
- Inline SVG, **no HTML wrapper**
- `<script>` tags are **stripped** ([index.html:1568](index.html#L1568)) — an animated SVG must use SMIL or CSS only

**Type `mermaid`**:
- Pure Mermaid syntax, **no backticks** around it, no Markdown block
- Theme `default`, `securityLevel: 'loose'`

**Type `markdown`**:
- Rendered via `marked.parse` — inline HTML works
- If `IsDoc=true`, the artefact appears in the "Documentation" chips of the AI panel

#### Common Constraints (all types)

- **Iframe sandbox** = `allow-scripts allow-same-origin`
  - No `top.location`, no cross-origin form submit
  - `localStorage` shared with host widget (readable by artefact)
- **Iframe srcdoc origin = `null`**
  - `fetch('/api/...')` (relative) **fails** — always use absolute URLs
  - Third-party services must return CORS `Access-Control-Allow-Origin: *` or `null`
- **No network outside CDN/CORS** except via `grist.docApi.*` which goes through the parent
- **Tailwind via CDN**: uses JIT mode, some dynamically generated classes (`bg-${color}-500`) don't work — prefer explicit classes

#### Generation Best Practices

The agent **must**:
- Always return a **complete and standalone** artefact — never `// ... rest of code`
- For `react`: export a component named `App`
- For `grist`: call `grist.ready({...})` before anything else
- Handle empty / loading / error states (degraded UI acceptable)
- Use `mappings` for Grist columns (with fallback `record[mappings.X] || record.X`)
- Prefer `<button>` over `<span style="cursor:pointer">` (a11y)
- Prefer CSS `:hover` over `onmouseover="..."` when possible

The agent **must never**:
- Invent an unlisted Grist API (`setOptions`, `selectedTable`, `window.app`, etc.)
- Emit `import` / `export` (Babel standalone doesn't resolve modules — the widget strips them but better not to generate them)
- Make a relative `fetch`
- Return partial or truncated code
- Mix multiple frameworks (jQuery + React, etc.)
- Use a React component name other than `App` or `Component`

#### Auto-Correction Mode (prefix to recognize)

When the widget sends a prompt that **starts with** `Fix the following errors:`, the agent should:
1. Read the `console` array from context (last 5 errors)
2. Identify the precise cause
3. Modify **only** what is broken — do not refactor healthy code
4. Preserve the existing approach / structure
5. Return the complete fixed code

The agent has 3 attempts max before the widget gives up. If it can't fix the issue, return `{ error: "..." }` rather than starting from scratch.

#### Security

- The webhook URL and API key are stored **in plaintext in the widget's `localStorage`** — readable by any rendered artefact (sandbox `same-origin`). Acceptable for single-user, document for multi-tenancy.
- The screenshot captures the rendered DOM — may send sensitive Grist data. Disable the 📸 button by default on confidential data.
- The agent must handle each request in isolation — do not maintain state that mixes users (the `apiKey` sent identifies the instance, not the request).

### Three Agent Patterns

#### Pattern A — Minimal Proxy (recommended to start)

An endpoint that calls Claude (or OpenAI) and returns an Artefactory JSON.
Implementable in n8n in 5 nodes or ~30 lines of Node.

> **Ready-to-use system prompt template**:
> [AGENT_SYSTEM_PROMPT.md](AGENT_SYSTEM_PROMPT.md) contains a complete, copyable system
> prompt that makes the agent respect all widget constraints (available Grist APIs,
> React env, auto-correction mode, anti-patterns).
> Substitute `{{MODE}}`, `{{DOCUMENTATION}}`, `{{GRIST_SCHEMA}}` on your webhook side.

**Node.js / Cloudflare Worker example (Anthropic SDK)**:

```js
import Anthropic from "@anthropic-ai/sdk";
const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

export default async function handler(req) {
  const body = await req.json();

  const sysPrompt = [
    `You are an assistant for Artefactory, a Grist IDE.`,
    `Artefact type to produce: ${body.mode}.`,
    body.documentation?.length
      ? `Contextual documentation:\n${body.documentation.map(d => `## ${d.name}\n${d.content}`).join("\n\n")}`
      : "",
    body.gristSchema
      ? `Available Grist schema:\n${JSON.stringify(body.gristSchema, null, 2)}`
      : "",
    `Response constraints:`,
    `- SINGLE code block in a \`\`\`${body.mode} code fence`,
    `- No import / export (Babel standalone doesn't resolve modules)`,
    `- For React: export a component named "App"`,
    `- For Grist: keep the grist.ready({ requiredAccess: ... }) call`
  ].filter(Boolean).join("\n\n");

  const userMsg = [
    `Current code (${body.codeContext}):`,
    "```",
    body.code,
    "```",
    "",
    body.console?.length ? `Console errors:\n${body.console.join("\n")}\n` : "",
    body.prompt
  ].join("\n");

  const reply = await anthropic.messages.create({
    model: "claude-sonnet-4-6",
    max_tokens: 8000,
    system: sysPrompt,
    messages: [
      ...(body.conversation || []).map(m => ({
        role: m.role === "assistant" ? "assistant" : "user",
        content: m.content
      })),
      { role: "user", content: userMsg }
    ]
  });

  const text = reply.content[0].text;
  const codeMatch = text.match(/```(?:\w+)?\n([\s\S]*?)```/);

  return Response.json({
    code:    codeMatch?.[1]?.trim(),
    message: text.replace(/```[\s\S]*?```/g, "").trim() || "Code generated"
  });
}
```

**Artefactory-side configuration**:
- Webhook URL: the public URL of your endpoint
- API Key: an arbitrary secret your endpoint will verify (header `Authorization: Bearer <secret>`)

**Equivalent n8n configuration**:
1. **Webhook** node (POST, response mode "When last node finishes")
2. **HTTP Request** or **Anthropic** node calling the Claude/GPT API with the system prompt above
3. **Code** node that parses the code fence
4. **Respond to Webhook** node with `{ code, message }`

#### Pattern B — Tooled Agent (tool use)

If you want fine-grained modifications (replace-lines, insert-at-line), you can let
Claude call structured tools and **post-process on the webhook side** to rebuild a
complete `code` to return. The current widget doesn't support partial patches — it
applies either everything or the selection.

Possible clean evolution: extend `handleAIResponse` ([index.html](index.html)) to
accept an `operations: [{ type: "replace", from, to, code }]` format. Not in place today.

#### Pattern C — Local Agent + MCP Grist

For an agent that can **read other tables** without going through the widget
(bypassing `gristSchema`), run Claude Agent SDK + a Grist MCP server locally,
expose an HTTP endpoint in front. More powerful but requires orchestration. Out of
scope for a minimal setup.

### Recommendation to Get Started

1. Deploy **Pattern A** on Cloudflare Workers / Vercel / n8n.
2. Paste the URL in the AI panel, configure a Bearer token on the webhook side.
3. Create a Markdown artefact `Conventions` with `IsDoc=true` that describes
   your in-house rules (React style, Grist conventions, etc.).
4. Select it before each AI request — it will be attached as context.
5. Test with a simple prompt ("Create a counter") then iterate on the webhook's system prompt.

Once stable, see if you want to extend to Pattern B (tool use) or C (MCP) based on your needs.

## Security

- **Store the API key on the server side**, not in Grist. The widget's `apiKey` field
  sends an `Authorization: Bearer` to YOUR webhook — it's up to you what to do with it.
- **Always protect the webhook** (Bearer token, IP allowlist, or both). Without
  protection, anyone who sees a screenshot of the widget with the URL can consume your quota.
- **The iframe sandbox is `allow-scripts allow-same-origin`**: a malicious artefact
  can access the host widget's localStorage (and thus read the configured token).
  Do not load un-audited artefacts in a shared environment.
- **The screenshot captures the rendered DOM**: if the artefact displays sensitive
  Grist data, it will be sent in the request. Disable the 📸 button by default on
  confidential data.

## Known Limitations

- No artefact versioning (each save overwrites).
- No sharing between Grist documents (each doc has its own `Artefacts` table).
- No nested iframe support (an artefact cannot embed another artefact). See [CLAUDE.md](CLAUDE.md) for patterns to bring back from the historical v12 if multi-artefact composition is desired.
- No AI response streaming.
- The conversation history sent to the webhook is truncated to 10 messages × 500 characters.
