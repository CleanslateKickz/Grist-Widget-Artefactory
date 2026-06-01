# Minimal Architecture Guide — Artefactory Runtime

This document describes **exactly** what needs to be reproduced in a minimal `app.html` to run the existing templates, without the Ace editor, without the AI assistant, and without Edit mode.

---

## 1. Execution Flow Overview

```
app.html (parent Grist widget)
  │
  ├─ 1. grist.ready({ requiredAccess: 'full' })
  ├─ 2. Load the "Artefacts" table via fetchTable()
  ├─ 3. Identify artefacts of type "app" → parse their Code as JSON manifest
  ├─ 4. Select an app → read manifest.routes
  ├─ 5. Build the sidebar (icons + labels from routes)
  ├─ 6. On route click:
  │     a. Find the corresponding artefact (route.artefact → id)
  │     b. Take its Code (raw HTML)
  │     c. Inject GRIST_BRIDGE_SCRIPT (replaces grist-plugin-api.js tag)
  │     d. Inject APP_RUNTIME_SCRIPT (window.app in <head>)
  │     e. Write HTML into iframe.srcdoc
  │     f. Connect GristBridgeParent to this iframe
  └─ 7. Listen for postMessage from iframe (navigation, events)
```

---

## 2. The 5 Essential Building Blocks

### BLOCK 1 — Loading Data from Grist

**Required table**: `Artefacts` with columns:
- `Nom` (Text), `Type` (Choice), `Code` (Text), `Dependencies` (Text)

**Columnar → object conversion** (same pattern as tasks_app):
```javascript
const TABLE_NAME = 'Artefacts';

async function loadArtefacts() {
    const data = await grist.docApi.fetchTable(TABLE_NAME);
    const items = [];
    const apps = [];
    const n = data.id ? data.id.length : 0;

    for (let i = 0; i < n; i++) {
        const art = {
            id: data.id[i],
            Nom: data.Nom?.[i] || '',
            Type: data.Type?.[i] || '',
            Code: data.Code?.[i] || '',
        };
        items.push(art);
        if (art.Type === 'app') apps.push(art);
    }

    state.artefacts = items;
    state.apps = apps;
}
```

Artefacts with `Type === 'app'` contain a JSON manifest in their `Code` column.

---

### BLOCK 2 — GristBridgeParent (parent side)

**Problem solved**: Sandboxed iframes cannot load `grist-plugin-api.js` from the parent. The bridge intercepts calls and relays them.

**Parent-side class** (in app.html) — **copy in full**:

```javascript
class GristBridgeParent {
    constructor() {
        this.currentIframe = null;
        this.callbacks = new Map();
        this.nextId = 1;
        window.addEventListener('message', this._handleMessage.bind(this));
    }

    setCurrentIframe(iframe) {
        this.callbacks.clear();
        this.currentIframe = iframe;
    }

    async _handleMessage(event) {
        const data = event.data;
        if (!data || data.type !== 'grist-bridge') return;
        if (!this.currentIframe || event.source !== this.currentIframe.contentWindow) return;

        try {
            let result;
            switch (data.action) {
                case 'ready': result = { success: true }; break;
                case 'apiCall': result = await this._handleApiCall(data); break;
                case 'registerCallback': result = await this._handleRegisterCallback(data); break;
                default: throw new Error('Unknown action');
            }
            this._sendResponse(data.callId, result, null);
        } catch (error) {
            this._sendResponse(data.callId, null, error.message);
        }
    }

    _sendResponse(callId, result, error) {
        if (!this.currentIframe) return;
        this.currentIframe.contentWindow.postMessage(
            { type: 'grist-bridge-response', callId, result, error }, '*'
        );
    }

    _sendCallback(callbackId, args) {
        if (!this.currentIframe) return;
        this.currentIframe.contentWindow.postMessage(
            { type: 'grist-bridge-callback', callbackId, args }, '*'
        );
    }

    async _handleApiCall(data) {
        const { path, args } = data;
        const parts = path.split('.');
        let target = grist;
        for (let i = 0; i < parts.length - 1; i++) {
            target = target[parts[i]];
            if (!target) throw new Error(`Invalid path: ${path}`);
        }
        const method = target[parts[parts.length - 1]];
        if (typeof method !== 'function') throw new Error(`${path} is not a function`);
        return await method.apply(target, args || []);
    }

    async _handleRegisterCallback(data) {
        const { callbackType } = data;
        const callbackId = 'cb_' + (this.nextId++);
        const self = this;

        if (callbackType === 'onRecords') {
            grist.onRecords((records, mappings) => self._sendCallback(callbackId, [records, mappings]));
        } else if (callbackType === 'onRecord') {
            grist.onRecord((record, mappings) => self._sendCallback(callbackId, [record, mappings]));
        }

        return { callbackId };
    }
}
```

**Message protocol**:
```
iframe → parent : { type: 'grist-bridge', action: 'apiCall', callId, path, args }
parent → iframe : { type: 'grist-bridge-response', callId, result, error }
parent → iframe : { type: 'grist-bridge-callback', callbackId, args }  (for onRecords/onRecord)
```

---

### BLOCK 3 — GRIST_BRIDGE_SCRIPT (iframe side, injected)

This script **replaces** the `<script src="grist-plugin-api.js">` tag in the artefact HTML. It creates a fake `window.grist` object that communicates with the parent via postMessage.

**Copy in full as a string**:

```javascript
const GRIST_BRIDGE_SCRIPT = `
(function() {
    const pending = new Map();
    const callbacks = new Map();
    let nextId = 1;

    window.addEventListener('message', (event) => {
        const data = event.data;
        if (data?.type === 'grist-bridge-response') {
            const p = pending.get(data.callId);
            if (p) {
                pending.delete(data.callId);
                data.error ? p.reject(new Error(data.error)) : p.resolve(data.result);
            }
        }
        if (data?.type === 'grist-bridge-callback') {
            const cb = callbacks.get(data.callbackId);
            if (cb) cb.apply(null, data.args || []);
        }
    });

    function send(action, payload) {
        return new Promise((resolve, reject) => {
            const callId = 'call_' + (nextId++);
            pending.set(callId, { resolve, reject });
            window.parent.postMessage({ type: 'grist-bridge', action, callId, ...payload }, '*');
            setTimeout(() => {
                if (pending.has(callId)) { pending.delete(callId); reject(new Error('Timeout')); }
            }, 30000);
        });
    }

    async function registerCallback(type, fn) {
        const result = await send('registerCallback', { callbackType: type });
        callbacks.set(result.callbackId, fn);
    }

    window.grist = {
        ready: (options) => send('ready', { options }),
        docApi: {
            listTables: () => send('apiCall', { path: 'docApi.listTables', args: [] }),
            fetchTable: (t) => send('apiCall', { path: 'docApi.fetchTable', args: [t] }),
            applyUserActions: (a) => send('apiCall', { path: 'docApi.applyUserActions', args: [a] })
        },
        onRecords: (cb) => registerCallback('onRecords', cb),
        onRecord: (cb) => registerCallback('onRecord', cb)
    };

    console.log('🌉 Grist Bridge ready');
})();
`;
```

**Injection** — replaces the Grist script tag with the bridge:
```javascript
function injectGristBridge(html) {
    return html.replace(
        /<script\s+src=["']https:\/\/docs\.getgrist\.com\/grist-plugin-api\.js["']\s*><\/script>/gi,
        '<script>' + GRIST_BRIDGE_SCRIPT + '<\/script>'
    );
}
```

---

### BLOCK 4 — APP_RUNTIME_SCRIPT (iframe side, injected)

Creates `window.app` in the iframe to enable:
- **Navigation**: `app.navigate('/batiments')` → postMessage to parent → `navigateToRoute()`
- **Events**: `app.emit('refresh', data)` → parent can broadcast to other iframes
- **Shared state**: `app.state`, `app.manifest`
- **Context detection**: templates use `typeof window.app !== 'undefined'` to know if they are in Artefactory

**Copy in full as a string**:

```javascript
const APP_RUNTIME_SCRIPT = `
(function() {
    const eventHandlers = new Map();

    window.app = {
        navigate: (path) => {
            window.parent.postMessage({ type: 'app-navigate', path }, '*');
        },
        getCurrentRoute: () => window.__APP_ROUTE__ || '/',

        emit: (event, data) => {
            window.parent.postMessage({ type: 'app-event', event, data }, '*');
        },
        on: (event, callback) => {
            if (!eventHandlers.has(event)) eventHandlers.set(event, []);
            eventHandlers.get(event).push(callback);
        },

        state: window.__APP_STATE__ || {},
        setState: (key, value) => {
            window.parent.postMessage({ type: 'app-setState', key, value }, '*');
        },

        manifest: window.__APP_MANIFEST__ || null,
        isEditMode: false
    };

    window.addEventListener('message', (event) => {
        const data = event.data;
        if (data?.type === 'app-event-broadcast') {
            const handlers = eventHandlers.get(data.event) || [];
            handlers.forEach(h => h(data.data));
        }
    });
})();
`;
```

**Injection** — inserts manifest + route + script before `</head>`:
```javascript
function injectAppRuntime(html, manifest, route) {
    const runtimeInit = `
        <script>
            window.__APP_MANIFEST__ = ${JSON.stringify(manifest)};
            window.__APP_ROUTE__ = "${route}";
            window.__APP_STATE__ = {};
            ${APP_RUNTIME_SCRIPT}
        <\/script>
    `;

    if (html.includes('</head>')) {
        return html.replace('</head>', runtimeInit + '</head>');
    }
    return runtimeInit + html;
}
```

---

### BLOCK 5 — Routing + Sidebar + Rendering

#### 5a. Manifest Parsing

When an app is selected, its `Code` is parsed as JSON:
```javascript
app.manifest = JSON.parse(app.Code);
```

Expected manifest structure (see `templates/manifest.json`):
```json
{
    "name": "App name",
    "icon": "🏛️",
    "layout": "sidebar",
    "theme": { "primary": "#16b378" },
    "routes": [
        { "path": "/", "label": "Home", "icon": "🏠", "artefact": 27 },
        { "path": "/batiments", "label": "Buildings", "icon": "🏢", "artefact": 28 }
    ]
}
```

Each `route.artefact` is a **record ID** in the Artefacts table.

#### 5b. Sidebar Construction

```javascript
function loadAppRuntime(app) {
    const manifest = app.manifest;

    // Logo + title
    document.getElementById('runtimeLogo').textContent = manifest.icon || '📦';
    document.getElementById('runtimeTitle').textContent = manifest.name || 'Application';

    // Build navigation items
    const nav = document.getElementById('runtimeNav');
    nav.innerHTML = '';
    (manifest.routes || []).forEach(route => {
        const item = document.createElement('div');
        item.className = 'nav-item' + (route.path === state.currentRoute ? ' active' : '');
        item.innerHTML = `
            <span class="nav-item-icon">${route.icon || '📄'}</span>
            <span class="nav-item-label">${route.label || route.path}</span>
        `;
        item.onclick = () => navigateToRoute(route.path);
        nav.appendChild(item);
    });

    // Load initial route
    navigateToRoute(state.currentRoute);
}
```

#### 5c. Navigation to a Route

```javascript
function navigateToRoute(path) {
    state.currentRoute = path;
    if (!state.currentApp?.manifest) return;

    const route = state.currentApp.manifest.routes?.find(r => r.path === path);

    // Update active state in sidebar
    document.querySelectorAll('.nav-item').forEach((item, i) => {
        item.classList.toggle('active', state.currentApp.manifest.routes[i]?.path === path);
    });

    // Breadcrumb
    document.getElementById('runtimeBreadcrumb').textContent = route?.label || path;

    // Find and render the artefact
    if (route?.artefact) {
        const artefact = state.artefacts.find(a => a.id === route.artefact);
        if (artefact) {
            renderInRuntime(artefact);
        }
    }
}
```

#### 5d. Rendering in the iframe

```javascript
function renderInRuntime(artefact) {
    const iframe = document.getElementById('runtimeIframe');
    let html = artefact.Code || '';

    // 1. Inject window.app (navigation, events, state)
    if (state.currentApp?.manifest) {
        html = injectAppRuntime(html, state.currentApp.manifest, state.currentRoute);
    }

    // 2. Replace grist-plugin-api.js with the bridge
    if (html.includes('grist-plugin-api.js')) {
        html = injectGristBridge(html);
    }

    // 3. Connect the parent bridge to this iframe
    state.gristBridge.setCurrentIframe(iframe);

    // 4. Inject the HTML
    iframe.srcdoc = html;
}
```

**Critical injection order**:
1. First `injectAppRuntime` (adds `window.__APP_MANIFEST__` + `window.app` in `<head>`)
2. Then `injectGristBridge` (replaces the `grist-plugin-api.js` tag with the bridge)
3. Then `setCurrentIframe` (routes bridge messages to this iframe)
4. Finally `iframe.srcdoc = html` (launches execution)

---

## 3. Listening for Parent Messages (onMessage)

The parent must listen for `postMessage` from iframes:

```javascript
window.addEventListener('message', function(e) {
    const d = e.data;
    if (!d || !d.type) return;

    // Navigation requested by a template (app.navigate('/path'))
    if (d.type === 'app-navigate') {
        navigateToRoute(d.path);
    }

    // Event emitted by a template (app.emit('event', data))
    if (d.type === 'app-event') {
        // Optional: broadcast to other iframes if multi-iframe
        console.log('App event:', d.event, d.data);
    }
});
```

---

## 4. Minimal Runtime HTML Structure

```html
<body>
    <!-- Sidebar + Content -->
    <div class="runtime-container" id="runtimeContainer">
        <div class="runtime-main">
            <!-- Sidebar -->
            <div class="runtime-sidebar" id="runtimeSidebar">
                <div class="sidebar-header">
                    <div class="sidebar-logo" id="runtimeLogo">📦</div>
                    <div class="sidebar-title" id="runtimeTitle">Application</div>
                </div>
                <nav class="sidebar-nav" id="runtimeNav">
                    <!-- Dynamically generated items -->
                </nav>
            </div>

            <!-- Content area -->
            <div class="runtime-content">
                <div class="runtime-toolbar">
                    <span id="runtimeBreadcrumb">Home</span>
                </div>
                <div class="runtime-frame">
                    <iframe class="runtime-iframe" id="runtimeIframe"
                            sandbox="allow-scripts allow-same-origin"></iframe>
                </div>
            </div>
        </div>
    </div>
</body>
```

---

## 5. Minimal Runtime CSS

```css
* { box-sizing: border-box; margin: 0; padding: 0; }
body { font-family: system-ui, sans-serif; min-height: 100vh; display: flex; flex-direction: column; overflow: hidden; }

.runtime-container { flex: 1; display: flex; flex-direction: column; overflow: hidden; }
.runtime-main { flex: 1; display: flex; overflow: hidden; }

/* Sidebar */
.runtime-sidebar { width: 220px; background: #fff; border-right: 1px solid #e2e8f0; display: flex; flex-direction: column; flex-shrink: 0; }
.sidebar-header { padding: 16px; border-bottom: 1px solid #e2e8f0; display: flex; align-items: center; gap: 10px; }
.sidebar-logo { width: 36px; height: 36px; border-radius: 10px; background: linear-gradient(135deg, #059669, #10b981); display: flex; align-items: center; justify-content: center; color: #fff; font-size: 20px; }
.sidebar-title { font-size: 14px; font-weight: 700; }
.sidebar-nav { flex: 1; overflow-y: auto; padding: 8px; }

/* Navigation items */
.nav-item { padding: 10px 12px; border-radius: 8px; cursor: pointer; display: flex; align-items: center; gap: 10px; font-size: 13px; transition: all 0.15s; margin-bottom: 4px; }
.nav-item:hover { background: #f1f5f9; }
.nav-item.active { background: #3b82f6; color: #fff; }
.nav-item-icon { font-size: 18px; flex-shrink: 0; }
.nav-item-label { white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }

/* Content */
.runtime-content { flex: 1; display: flex; flex-direction: column; overflow: hidden; background: #f8fafc; }
.runtime-toolbar { padding: 8px 16px; background: #fff; border-bottom: 1px solid #e2e8f0; font-size: 13px; color: #64748b; }
.runtime-frame { flex: 1; overflow: hidden; position: relative; }
.runtime-iframe { width: 100%; height: 100%; border: none; background: #fff; }
```

---

## 6. Minimal Initialization Sequence

```javascript
const state = {
    artefacts: [],
    apps: [],
    currentApp: null,
    currentRoute: '/',
    gristBridge: null,
};

async function init() {
    // 1. Create the bridge
    state.gristBridge = new GristBridgeParent();

    // 2. Listen for iframe messages
    window.addEventListener('message', onMessage);

    // 3. Connect to Grist
    grist.ready({ requiredAccess: 'full' });

    // 4. Load artefacts
    await loadArtefacts();

    // 5. Auto-select the first app
    if (state.apps.length > 0) {
        const app = state.apps[0];
        try { app.manifest = JSON.parse(app.Code); } catch { app.manifest = { routes: [] }; }
        state.currentApp = app;
        loadAppRuntime(app);
    }
}

init();
```

---

## 7. What Templates Expect

Each template (Home, batiments, etc.) relies on these two injected APIs:

### Grist API (via bridge)
```javascript
// These calls go through the bridge postMessage → parent → real grist API
await grist.docApi.fetchTable('Batiments');
await grist.docApi.applyUserActions([['UpdateRecord', 'Batiments', id, data]]);
```

### App API (via runtime script)
```javascript
// Context detection
const appContext = {
    isArtefactory: typeof window.app !== 'undefined' && window.app.navigate,
    app: window.app || { navigate: () => {}, emit: () => {}, on: () => {} }
};

// Navigation (parent interprets and changes route)
appContext.app.navigate('/batiments');

// Inter-widget events
appContext.app.emit('batiment-selected', { id: 42 });
appContext.app.on('batiment-selected', (data) => { ... });

// Access to manifest
const manifest = window.app?.manifest;
```

---

## 8. Summary: Files and Dependencies for the Minimal Runtime

### Single file to create: `app.html`
Contains:
1. **CSS** (~40 lines): sidebar + content layout
2. **HTML** (~20 lines): sidebar-header, sidebar-nav, iframe
3. **JS** (~200 lines):
   - `GristBridgeParent` (class, ~70 lines)
   - `GRIST_BRIDGE_SCRIPT` (injected string, ~40 lines)
   - `APP_RUNTIME_SCRIPT` (injected string, ~30 lines)
   - `injectGristBridge()` + `injectAppRuntime()` (~15 lines)
   - `loadArtefacts()` (~20 lines)
   - `loadAppRuntime()` + `navigateToRoute()` + `renderInRuntime()` (~40 lines)
   - `onMessage()` (~10 lines)
   - `init()` (~15 lines)

### Single CDN dependency
```html
<script src="https://docs.getgrist.com/grist-plugin-api.js"></script>
```
(Tailwind, Chart.js etc. are loaded by the templates themselves, not by app.html)

### Required Grist Table
`Artefacts` with at minimum: `Nom`, `Type`, `Code`

### Existing templates (unchanged)
The files in `templates/` will work as-is because the minimal runtime exposes exactly the same APIs (`window.grist` via bridge + `window.app` via runtime script).
