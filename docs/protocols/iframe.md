# iframe Sandbox

The iframe sandbox executes BTCP tools in an isolated browsing context using sandboxed iframes.

## Overview

Sandboxed iframes provide DOM isolation with configurable restrictions. This is useful for tools that need direct DOM manipulation within an isolated context.

## Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                       Main Document                             │
│                                                                  │
│  ┌─────────────┐    ┌─────────────┐                            │
│  │  Web App    │    │ BTCP Client │                            │
│  │             │    │             │                            │
│  └─────────────┘    └──────┬──────┘                            │
│                            │                                    │
│  ┌─────────────────────────▼────────────────────────────────┐  │
│  │                    Sandbox iframe                         │  │
│  │  sandbox="allow-scripts"                                  │  │
│  │  csp="default-src 'none'; script-src 'unsafe-inline'"    │  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────────────────────────┐ │  │
│  │  │                 Isolated Document                    │ │  │
│  │  │                                                      │ │  │
│  │  │  ┌──────────────────────────────────────────────┐  │ │  │
│  │  │  │            Capability-Gated API              │  │ │  │
│  │  │  │                                              │  │ │  │
│  │  │  │  ┌─────────────┐  ┌─────────────────────┐   │  │ │  │
│  │  │  │  │ DOM Access  │  │ postMessage Bridge  │   │  │ │  │
│  │  │  │  └─────────────┘  └─────────────────────┘   │  │ │  │
│  │  │  └──────────────────────────────────────────────┘  │ │  │
│  │  │                                                      │ │  │
│  │  │  ┌──────────────────────────────────────────────┐  │ │  │
│  │  │  │              Tool Execution                  │  │ │  │
│  │  │  └──────────────────────────────────────────────┘  │ │  │
│  │  └─────────────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

## Isolation Properties

### Browsing Context Isolation
- Separate document and window objects
- Isolated JavaScript execution context
- No direct access to parent document

### Sandbox Restrictions
- Configurable via `sandbox` attribute
- Can restrict forms, scripts, popups, etc.
- Origin isolation available

### Communication
- `postMessage()` for cross-frame communication
- Structured clone for data transfer
- Origin validation required

## Sandbox Flags

| Flag | Effect |
|------|--------|
| `allow-scripts` | Enables JavaScript execution |
| `allow-same-origin` | Allows same-origin access (use carefully) |
| `allow-forms` | Enables form submission |
| `allow-popups` | Allows popup windows |
| `allow-modals` | Allows modal dialogs |

**Recommended Configuration:**

```html
<iframe
  sandbox="allow-scripts"
  src="about:blank"
></iframe>
```

**Warning:** Never use `allow-same-origin` with `allow-scripts` for untrusted code, as this allows sandbox escape.

## Implementation

### iframe Manager

```javascript
class IframeManager {
  constructor() {
    this.iframes = new Map();
  }

  async createSandbox(capabilities) {
    const iframe = document.createElement('iframe');

    // Minimal sandbox permissions
    iframe.sandbox = 'allow-scripts';

    // Content Security Policy
    const csp = this.buildCSP(capabilities);

    // Create blob URL with bootstrap code
    const content = this.buildIframeContent(csp);
    iframe.src = URL.createObjectURL(
      new Blob([content], { type: 'text/html' })
    );

    // Hide iframe
    iframe.style.display = 'none';

    document.body.appendChild(iframe);

    // Wait for iframe to load
    await new Promise(resolve => {
      iframe.onload = resolve;
    });

    return iframe;
  }

  buildCSP(capabilities) {
    const directives = [
      "default-src 'none'",
      "script-src 'unsafe-inline'",
    ];

    if (capabilities.some(c => c.startsWith('network:fetch'))) {
      const origins = this.getAllowedOrigins(capabilities);
      directives.push(`connect-src ${origins.join(' ')}`);
    }

    return directives.join('; ');
  }

  buildIframeContent(csp) {
    return `
      <!DOCTYPE html>
      <html>
      <head>
        <meta http-equiv="Content-Security-Policy" content="${csp}">
      </head>
      <body>
        <script>
          ${iframeBootstrap}
        </script>
      </body>
      </html>
    `;
  }
}
```

### iframe Bootstrap

```javascript
const iframeBootstrap = `
  // Listen for tool execution requests
  window.addEventListener('message', async (event) => {
    // Validate origin
    if (event.origin !== expectedOrigin) return;

    const { id, type, toolCode, args, capabilities } = event.data;

    if (type !== 'execute') return;

    try {
      // Create capability-gated API
      const api = createCapabilityAPI(capabilities, event.source);

      // Execute tool
      const toolFunction = new Function('btcp', 'args', toolCode);
      const result = await toolFunction(api, args);

      // Send result back
      event.source.postMessage({
        id,
        type: 'result',
        result
      }, event.origin);

    } catch (error) {
      event.source.postMessage({
        id,
        type: 'error',
        error: {
          code: 'EXECUTION_ERROR',
          message: error.message
        }
      }, event.origin);
    }
  });

  function createCapabilityAPI(capabilities, parent) {
    const api = {};

    // For iframe sandbox, we can access our own DOM
    // but need to proxy to parent for main document access
    if (capabilities.includes('dom:read')) {
      api.parentDOM = {
        querySelector: (selector) => {
          return sendToParent(parent, 'dom.querySelector', { selector });
        }
      };
    }

    return api;
  }

  async function sendToParent(parent, method, params) {
    return new Promise((resolve, reject) => {
      const id = crypto.randomUUID();

      const handler = (event) => {
        if (event.data.id !== id) return;
        window.removeEventListener('message', handler);

        if (event.data.error) {
          reject(new Error(event.data.error.message));
        } else {
          resolve(event.data.result);
        }
      };

      window.addEventListener('message', handler);
      parent.postMessage({ id, method, params }, '*');
    });
  }
`;
```

### Tool Execution

```javascript
class IframeSandbox {
  async executeTool(toolCode, args, capabilities) {
    const iframe = await this.manager.createSandbox(capabilities);

    return new Promise((resolve, reject) => {
      const id = crypto.randomUUID();
      const timeout = setTimeout(() => {
        cleanup();
        reject(new BTCPTimeoutError('Tool execution timed out'));
      }, 30000);

      const cleanup = () => {
        clearTimeout(timeout);
        window.removeEventListener('message', handler);
        iframe.remove();
      };

      const handler = (event) => {
        if (event.data.id !== id) return;

        cleanup();

        if (event.data.type === 'error') {
          reject(new BTCPExecutionError(event.data.error));
        } else {
          resolve(event.data.result);
        }
      };

      window.addEventListener('message', handler);

      iframe.contentWindow.postMessage({
        id,
        type: 'execute',
        toolCode,
        args,
        capabilities
      }, '*');
    });
  }
}
```

## Use Cases

### DOM Rendering Tools

Tools that need to render and manipulate DOM elements:

```javascript
// Tool that renders markdown and extracts text
async function renderMarkdown({ markdown }) {
  const container = document.createElement('div');
  container.innerHTML = markdownToHTML(markdown);

  // Can manipulate DOM within iframe
  const headings = container.querySelectorAll('h1, h2, h3');

  return {
    html: container.innerHTML,
    headings: Array.from(headings).map(h => h.textContent)
  };
}
```

### Isolated Computation with DOM APIs

```javascript
// Tool that uses DOMParser (unavailable in Workers)
async function parseXML({ xmlString }) {
  const parser = new DOMParser();
  const doc = parser.parseFromString(xmlString, 'application/xml');

  // Extract data from parsed XML
  const items = doc.querySelectorAll('item');
  return Array.from(items).map(item => ({
    title: item.querySelector('title')?.textContent,
    link: item.querySelector('link')?.textContent
  }));
}
```

## Configuration

```json
{
  "config": {
    "sandbox": "iframe",
    "iframeOptions": {
      "sandbox": "allow-scripts",
      "csp": {
        "default-src": "'none'",
        "script-src": "'unsafe-inline'"
      }
    }
  }
}
```

## Security Considerations

1. **Origin Validation**: Always validate `event.origin` in message handlers
2. **No allow-same-origin**: Never combine with `allow-scripts`
3. **CSP**: Use strict Content Security Policy
4. **Input Validation**: Validate all messages before processing
5. **Cleanup**: Remove iframes after use to prevent memory leaks

## Comparison with Web Workers

| Aspect | iframe | Web Worker |
|--------|--------|------------|
| DOM Access | Own document | None |
| Thread | Main thread | Separate thread |
| DOM APIs | Full | Limited |
| Isolation | Browsing context | Thread |
| Performance | Good | Better |

## Browser Support

| Browser | Support |
|---------|---------|
| Chrome | ✅ Full |
| Firefox | ✅ Full |
| Safari | ✅ Full |
| Edge | ✅ Full |

## Related

- [Protocols Overview](./index.md)
- [Web Worker Sandbox](./worker.md)
- [SES Sandbox](./ses.md)
- [Security Model](../security.md)
