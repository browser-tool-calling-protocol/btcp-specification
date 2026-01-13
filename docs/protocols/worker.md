# Web Worker Sandbox

The Web Worker sandbox executes BTCP tools in a separate thread using the [Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API).

## Overview

Web Workers provide thread-based isolation with message-passing communication. This is the default and most widely compatible sandbox option.

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│                      Main Thread                            │
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌────────────────┐  │
│  │  Web App    │    │ BTCP Client │    │ Worker Manager │  │
│  │             │    │             │    │                │  │
│  │             │    │             │───►│ postMessage()  │  │
│  │             │    │             │◄───│ onmessage      │  │
│  └─────────────┘    └─────────────┘    └───────┬────────┘  │
│                                                 │           │
└─────────────────────────────────────────────────┼───────────┘
                                                  │
                    ┌─────────────────────────────▼───────────┐
                    │              Worker Thread               │
                    │                                          │
                    │  ┌────────────────────────────────────┐ │
                    │  │        Capability-Gated API         │ │
                    │  │                                    │ │
                    │  │  ┌──────────┐  ┌──────────────┐   │ │
                    │  │  │ DOM Proxy│  │Storage Proxy │   │ │
                    │  │  └──────────┘  └──────────────┘   │ │
                    │  │                                    │ │
                    │  │  ┌────────────────────────────┐   │ │
                    │  │  │      Tool Execution        │   │ │
                    │  │  └────────────────────────────┘   │ │
                    │  └────────────────────────────────────┘ │
                    │                                          │
                    └──────────────────────────────────────────┘
```

## Isolation Properties

### Thread Isolation
- Tools run in a separate OS thread
- No shared memory with main thread (without SharedArrayBuffer)
- Crashes don't affect main thread

### API Restrictions
- No direct DOM access
- No `window` object
- No `document` object
- Limited to Worker global scope APIs

### Communication
- Structured clone algorithm for message passing
- No direct object references
- Async message-based API proxying

## Implementation

### Worker Manager

```javascript
class WorkerManager {
  constructor() {
    this.workers = new Map();
  }

  async executeInWorker(toolCode, args, capabilities) {
    const worker = new Worker(
      URL.createObjectURL(new Blob([workerBootstrap], { type: 'application/javascript' }))
    );

    return new Promise((resolve, reject) => {
      const timeout = setTimeout(() => {
        worker.terminate();
        reject(new BTCPTimeoutError('Tool execution timed out'));
      }, 30000);

      worker.onmessage = (event) => {
        clearTimeout(timeout);
        worker.terminate();

        if (event.data.error) {
          reject(new BTCPExecutionError(event.data.error));
        } else {
          resolve(event.data.result);
        }
      };

      worker.onerror = (error) => {
        clearTimeout(timeout);
        worker.terminate();
        reject(new BTCPExecutionError(error.message));
      };

      worker.postMessage({
        toolCode,
        args,
        capabilities
      });
    });
  }
}
```

### Worker Bootstrap

```javascript
// worker-bootstrap.js
const workerBootstrap = `
self.onmessage = async (event) => {
  const { toolCode, args, capabilities } = event.data;

  // Create capability-gated API
  const api = createCapabilityAPI(capabilities);

  try {
    // Create isolated function scope
    const toolFunction = new Function('btcp', 'args', toolCode);
    const result = await toolFunction(api, args);

    self.postMessage({ result });
  } catch (error) {
    self.postMessage({
      error: {
        code: 'EXECUTION_ERROR',
        message: error.message,
        stack: error.stack
      }
    });
  }
};

function createCapabilityAPI(capabilities) {
  return {
    dom: capabilities.includes('dom:read') ? createDOMProxy() : null,
    storage: createStorageProxy(capabilities),
    // ... other capability-gated APIs
  };
}
`;
```

### DOM Proxy

Since Workers cannot access DOM directly, operations are proxied through message passing:

```javascript
// In worker
function createDOMProxy() {
  return {
    querySelector: async (selector) => {
      return await sendToMain('dom.querySelector', { selector });
    },
    querySelectorAll: async (selector) => {
      return await sendToMain('dom.querySelectorAll', { selector });
    },
    getTextContent: async (selector) => {
      return await sendToMain('dom.getTextContent', { selector });
    }
  };
}

// In main thread
function handleDOMRequest(method, params) {
  switch (method) {
    case 'dom.querySelector':
      const element = document.querySelector(params.selector);
      return serializeElement(element);

    case 'dom.getTextContent':
      const el = document.querySelector(params.selector);
      return el?.textContent ?? null;
  }
}
```

## Capability Enforcement

Capabilities are enforced by only exposing permitted APIs:

```javascript
function createCapabilityAPI(capabilities) {
  const api = {};

  // DOM capabilities
  if (capabilities.includes('dom:read')) {
    api.dom = {
      querySelector: async (s) => sendToMain('dom.querySelector', { selector: s }),
      getTextContent: async (s) => sendToMain('dom.getTextContent', { selector: s }),
    };
  }

  if (capabilities.includes('dom:write')) {
    api.dom = {
      ...api.dom,
      setTextContent: async (s, v) => sendToMain('dom.setTextContent', { selector: s, value: v }),
      click: async (s) => sendToMain('dom.click', { selector: s }),
    };
  }

  // Storage capabilities
  if (capabilities.includes('storage:local:read')) {
    api.localStorage = {
      getItem: async (k) => sendToMain('storage.local.getItem', { key: k }),
    };
  }

  if (capabilities.includes('storage:local:write')) {
    api.localStorage = {
      ...api.localStorage,
      setItem: async (k, v) => sendToMain('storage.local.setItem', { key: k, value: v }),
    };
  }

  return api;
}
```

## Configuration

```json
{
  "config": {
    "sandbox": "worker",
    "workerOptions": {
      "type": "module",
      "name": "btcp-tool-worker"
    }
  }
}
```

## Limitations

| Limitation | Workaround |
|------------|------------|
| No synchronous DOM access | Async proxy API |
| Message passing overhead | Batch operations |
| No SharedArrayBuffer (some browsers) | Structured clone |
| Worker startup time | Worker pooling |

## Security Considerations

1. **Code Injection**: Tool code is executed via `new Function()`. Ensure proper input validation.
2. **Message Validation**: All messages from worker should be validated before processing.
3. **Resource Limits**: Implement timeouts and memory limits to prevent DoS.

## Browser Support

| Browser | Support |
|---------|---------|
| Chrome | ✅ Full |
| Firefox | ✅ Full |
| Safari | ✅ Full |
| Edge | ✅ Full |

## Related

- [Protocols Overview](./index.md)
- [iframe Sandbox](./iframe.md)
- [SES Sandbox](./ses.md)
- [Security Model](../security.md)
