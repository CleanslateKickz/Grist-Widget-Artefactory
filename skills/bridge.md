# GristBridge - Sandboxed Iframe Communication

## Problem

Grist widgets run in sandboxed iframes. A widget that loads content into a **sub-iframe** (e.g. Artefactory) cannot directly access the Grist API from that sub-iframe.

```
┌─────────────────────────────────────────────┐
│ Grist (parent)                              │
│  └── Widget iframe (has Grist API access)    │
│       └── Sub-iframe (NO API access)        │ ← Problem!
└─────────────────────────────────────────────┘
```

## Solution: GristBridge

The bridge intercepts API calls from sub-iframes and routes them to the parent that has access to Grist.

```
Sub-iframe              Widget (parent)           Grist
    │                         │                      │
    ├─ postMessage ──────────►│                      │
    │  {grist-bridge, fetch}  │── grist.docApi ─────►│
    │                         │◄── data ─────────────┤
    │◄── postMessage ─────────┤                      │
    │   {result: data}        │                      │
```

---

## Parent-side Implementation

### GristBridgeParent class
```javascript
class GristBridgeParent {
    constructor() {
        this.callbackSources = new Map();
        this.nextId = 1;
        window.addEventListener('message', this._handleMessage.bind(this));
    }

    async _handleMessage(event) {
        const data = event.data;
        if (!data || data.type !== 'grist-bridge') return;

        const source = event.source;
        if (!source) return;

        const { action, args, callbackId } = data;

        try {
            let result;
            switch (action) {
                case 'fetchTable':
                    result = await grist.docApi.fetchTable(args.tableName);
                    break;
                case 'applyUserActions':
                    result = await grist.docApi.applyUserActions(args.actions);
                    break;
                case 'listTables':
                    result = await grist.docApi.listTables();
                    break;
                default:
                    throw new Error(`Unknown action: ${action}`);
            }

            source.postMessage({
                type: 'grist-bridge-response',
                callbackId,
                result
            }, '*');
        } catch (error) {
            source.postMessage({
                type: 'grist-bridge-response',
                callbackId,
                error: error.message
            }, '*');
        }
    }
}

// Initialize the bridge
const gristBridge = new GristBridgeParent();
```

---

## Script injected into sub-iframes

### GRIST_BRIDGE_SCRIPT
```javascript
const GRIST_BRIDGE_SCRIPT = `
(function() {
    const callbacks = new Map();
    let nextId = 1;

    window.addEventListener('message', function(event) {
        const data = event.data;
        if (!data || data.type !== 'grist-bridge-response') return;

        const cb = callbacks.get(data.callbackId);
        if (cb) {
            callbacks.delete(data.callbackId);
            if (data.error) {
                cb.reject(new Error(data.error));
            } else {
                cb.resolve(data.result);
            }
        }
    });

    function bridgeCall(action, args) {
        return new Promise(function(resolve, reject) {
            const callbackId = 'cb_' + (nextId++);
            callbacks.set(callbackId, { resolve, reject });

            window.parent.postMessage({
                type: 'grist-bridge',
                action: action,
                args: args,
                callbackId: callbackId
            }, '*');

            // Timeout
            setTimeout(function() {
                if (callbacks.has(callbackId)) {
                    callbacks.delete(callbackId);
                    reject(new Error('Bridge timeout'));
                }
            }, 30000);
        });
    }

    // API compatible with grist.docApi
    window.grist = {
        ready: function() {},
        docApi: {
            fetchTable: function(tableName) {
                return bridgeCall('fetchTable', { tableName: tableName });
            },
            applyUserActions: function(actions) {
                return bridgeCall('applyUserActions', { actions: actions });
            },
            listTables: function() {
                return bridgeCall('listTables', {});
            }
        }
    };
})();
`;
```

---

## Injection into the iframe

### During rendering
```javascript
function renderInIframe(iframe, htmlContent) {
    // Inject the bridge before the content
    const bridgedHtml = htmlContent.replace(
        '<head>',
        `<head><script>${GRIST_BRIDGE_SCRIPT}</script>`
    );

    iframe.srcdoc = bridgedHtml;
}
```

### Complete pattern
```javascript
function createSandboxedWidget(container, html) {
    const iframe = document.createElement('iframe');
    iframe.sandbox = 'allow-scripts allow-same-origin';
    iframe.style.cssText = 'width:100%;border:none;';

    // Inject the bridge
    const injectedHtml = `
        <!DOCTYPE html>
        <html>
        <head>
            <script>${GRIST_BRIDGE_SCRIPT}</script>
        </head>
        <body>
            ${html}
        </body>
        </html>
    `;

    iframe.srcdoc = injectedHtml;
    container.appendChild(iframe);

    return iframe;
}
```

---

## Nested iframes (3 levels)

### Problem
For a level 3 iframe, `window.parent` points to level 2, not to the main widget.

```
Grist
└── Widget (level 1) ← has Grist API
    └── Component (level 2)
        └── Sub-component (level 3) ← must reach level 1
```

### Solution: Retarget
Before injecting the HTML, replace `window.parent` calls:

```javascript
function prepareNestedHtml(html) {
    // Retarget to grandparent
    return html.replace(
        /window\.parent\.postMessage/g,
        'window.parent.parent.postMessage'
    );
}
```

### Complete pattern for component inclusion
```javascript
const AUTORESIZE_SCRIPT = `
<script>
(function() {
    function notifyHeight() {
        var height = document.body.scrollHeight;
        window.parent.postMessage({ type: 'iframe-resize', height: height }, '*');
    }
    window.addEventListener('load', notifyHeight);
    new MutationObserver(notifyHeight).observe(document.body, {
        childList: true, subtree: true
    });
})();
</script>
`;

function prepareNestedHtml(html) {
    // 1. Retarget the bridge to the parent's parent
    let prepared = html.replace(
        /window\.parent\.postMessage/g,
        'window.parent.parent.postMessage'
    );

    // 2. Add the resize script (stays on immediate parent)
    prepared = prepared.replace('</body>', AUTORESIZE_SCRIPT + '</body>');

    return prepared;
}
```

---

## Resize Handling

### Parent side (listen for resize messages)
```javascript
window.addEventListener('message', (event) => {
    if (event.data?.type === 'iframe-resize') {
        const iframe = findIframeBySource(event.source);
        if (iframe) {
            iframe.style.height = event.data.height + 'px';
        }
    }
});

function findIframeBySource(source) {
    return Array.from(document.querySelectorAll('iframe'))
        .find(iframe => iframe.contentWindow === source);
}
```

### Iframe side (notify resize)
```javascript
function notifyResize() {
    const height = document.documentElement.scrollHeight;
    window.parent.postMessage({
        type: 'iframe-resize',
        height: height
    }, '*');
}

// Observe changes
new ResizeObserver(notifyResize).observe(document.body);
window.addEventListener('load', notifyResize);
```

---

## Security

### Recommended sandbox attribute
```html
<iframe sandbox="allow-scripts allow-same-origin"></iframe>
```

| Attribute | Effect |
|----------|-------|
| `allow-scripts` | Allows JavaScript execution |
| `allow-same-origin` | Required for postMessage |

### Message validation
```javascript
window.addEventListener('message', (event) => {
    // Ignore messages without a known type
    if (!event.data?.type) return;

    // Validate the type
    const validTypes = ['grist-bridge', 'grist-bridge-response', 'iframe-resize'];
    if (!validTypes.includes(event.data.type)) return;

    // Process...
});
```
