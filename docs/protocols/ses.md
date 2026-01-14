# SES Sandbox

The SES (Secure ECMAScript) sandbox executes BTCP tools using hardened JavaScript compartments with frozen intrinsics.

## Overview

SES provides the highest level of JavaScript-based isolation by freezing all intrinsic objects (Object, Array, Function, etc.) and creating isolated compartments with controlled global access.

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         Main Thread                               │
│                                                                    │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │                    SES Lockdown                            │   │
│  │                                                            │   │
│  │  ┌────────────────────────────────────────────────────┐   │   │
│  │  │              Frozen Intrinsics                      │   │   │
│  │  │  Object ❄️  Array ❄️  Function ❄️  Promise ❄️      │   │   │
│  │  └────────────────────────────────────────────────────┘   │   │
│  │                                                            │   │
│  │  ┌────────────────────────────────────────────────────┐   │   │
│  │  │               Compartment Pool                      │   │   │
│  │  │                                                     │   │   │
│  │  │  ┌──────────────────┐  ┌──────────────────┐       │   │   │
│  │  │  │  Compartment A   │  │  Compartment B   │       │   │   │
│  │  │  │                  │  │                  │       │   │   │
│  │  │  │  globals: {      │  │  globals: {      │       │   │   │
│  │  │  │    btcp: {...}   │  │    btcp: {...}   │       │   │   │
│  │  │  │  }               │  │  }               │       │   │   │
│  │  │  │                  │  │                  │       │   │   │
│  │  │  │  Tool Instance   │  │  Tool Instance   │       │   │   │
│  │  │  └──────────────────┘  └──────────────────┘       │   │   │
│  │  │                                                     │   │   │
│  │  └────────────────────────────────────────────────────┘   │   │
│  │                                                            │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌─────────────┐    ┌─────────────┐                              │
│  │  Web App    │    │ BTCP Client │                              │
│  └─────────────┘    └─────────────┘                              │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

## Isolation Properties

### Frozen Intrinsics
All built-in JavaScript objects are deeply frozen:
- No prototype pollution possible
- No modification of Object.prototype, Array.prototype, etc.
- Immutable standard library

### Controlled Globals
Compartments only expose explicitly provided globals:
- No `window`, `document`, `fetch` unless provided
- No `eval`, `Function` constructor escapes
- Complete control over available APIs

### Deterministic Execution
- Reproducible behavior across runs
- No hidden state leakage
- Predictable security properties

## Implementation

### SES Setup

```javascript
import 'ses';

// Lockdown must be called before any compartments
lockdown({
  // Error taming for better debugging
  errorTaming: 'unsafe',

  // Math.random is deterministic in compartments
  mathTaming: 'unsafe',

  // Date.now returns real time
  dateTaming: 'unsafe',

  // Console is available
  consoleTaming: 'unsafe',

  // Override mistakes throw
  overrideTaming: 'severe'
});
```

### Compartment Manager

```javascript
class SESManager {
  constructor() {
    this.compartments = new Map();
  }

  async executeTool(toolCode, args, capabilities) {
    // Create capability-gated endowments
    const endowments = this.createEndowments(capabilities);

    // Create isolated compartment
    const compartment = new Compartment({
      ...endowments,
      // No global access except what we provide
      __options__: true
    });

    try {
      // Evaluate tool code in compartment
      const toolFunction = compartment.evaluate(`(${toolCode})`);

      // Execute with provided arguments
      const result = await toolFunction(args);

      return result;

    } catch (error) {
      throw new BTCPExecutionError({
        code: 'EXECUTION_ERROR',
        message: error.message,
        stack: error.stack
      });
    }
  }

  createEndowments(capabilities) {
    const endowments = {
      // Safe built-ins
      console: harden(console),
      JSON: harden(JSON),
      Math: harden(Math),

      // BTCP API
      btcp: harden(this.createBTCPAPI(capabilities))
    };

    return harden(endowments);
  }

  createBTCPAPI(capabilities) {
    const api = {};

    if (capabilities.includes('dom:read')) {
      api.dom = {
        querySelector: async (selector) => {
          const element = document.querySelector(selector);
          return this.serializeElement(element);
        },
        getTextContent: async (selector) => {
          const element = document.querySelector(selector);
          return element?.textContent ?? null;
        }
      };
    }

    if (capabilities.includes('dom:write')) {
      api.dom = {
        ...api.dom,
        setTextContent: async (selector, value) => {
          const element = document.querySelector(selector);
          if (element) element.textContent = value;
          return !!element;
        }
      };
    }

    if (capabilities.includes('storage:local:read')) {
      api.localStorage = {
        getItem: (key) => localStorage.getItem(key)
      };
    }

    return api;
  }

  serializeElement(element) {
    if (!element) return null;

    return harden({
      tagName: element.tagName,
      id: element.id,
      className: element.className,
      textContent: element.textContent,
      attributes: Object.fromEntries(
        Array.from(element.attributes).map(a => [a.name, a.value])
      )
    });
  }
}
```

### Using Harden

All objects passed to or from compartments should be hardened:

```javascript
// Harden makes objects deeply immutable
const safeResult = harden({
  data: result.data,
  metadata: {
    timestamp: Date.now(),
    source: 'btcp-tool'
  }
});

// Hardened objects cannot be modified
safeResult.data = 'new value';  // Throws in strict mode
```

## Tool Development

### Writing SES-Compatible Tools

```javascript
// Tool code is evaluated in compartment
async function getPageInfo(args) {
  // Access provided APIs through btcp object
  const title = await btcp.dom.getTextContent('title');
  const heading = await btcp.dom.getTextContent('h1');

  // Standard objects work normally (but are frozen)
  const result = {
    title,
    heading,
    url: args.includeUrl ? window.location?.href : null
  };

  return result;
}
```

### Restrictions

Tools running in SES cannot:
- Modify built-in prototypes
- Access `window`, `document` directly
- Use `eval()` or `Function()` constructor
- Access anything not in endowments

```javascript
// These will fail in SES
Object.prototype.foo = 'bar';  // Frozen, throws error
Array.prototype.customMethod = () => {};  // Frozen
window.alert('hello');  // window not available
eval('code');  // eval not available
```

## Configuration

```json
{
  "config": {
    "sandbox": "ses",
    "sesOptions": {
      "errorTaming": "safe",
      "mathTaming": "safe",
      "dateTaming": "safe"
    }
  }
}
```

## Security Benefits

| Threat | Mitigation |
|--------|------------|
| Prototype pollution | Frozen intrinsics |
| Global access | Controlled endowments |
| Code injection | No eval/Function |
| State leakage | Compartment isolation |
| Side channels | Deterministic execution |

## Performance Characteristics

| Aspect | Performance |
|--------|-------------|
| Startup (first lockdown) | ~50-100ms |
| Compartment creation | ~1-5ms |
| Code evaluation | Native speed |
| Memory overhead | Minimal |

## Comparison

| Feature | SES | Web Worker | iframe |
|---------|-----|------------|--------|
| Thread | Main | Separate | Main |
| DOM Access | Proxied | Proxied | Own |
| Prototype Safety | ✅ Frozen | ❌ Mutable | ❌ Mutable |
| Global Control | ✅ Complete | ⚠️ Partial | ⚠️ Partial |
| Setup Cost | Low | Medium | Medium |

## Browser Support

| Browser | Support |
|---------|---------|
| Chrome | ✅ Full |
| Firefox | ✅ Full |
| Safari | ✅ Full |
| Edge | ✅ Full |

**Note:** SES requires the [ses](https://www.npmjs.com/package/ses) shim package.

## Related Resources

- [SES GitHub Repository](https://github.com/endojs/endo/tree/master/packages/ses)
- [Hardened JavaScript](https://hardenedjs.org/)
- [Agoric SES Documentation](https://docs.agoric.com/guides/js-programming/hardened-js.html)

## Related

- [Protocols Overview](./index.md)
- [Web Worker Sandbox](./worker.md)
- [WebAssembly Sandbox](./wasm.md)
- [Security Model](../security.md)
