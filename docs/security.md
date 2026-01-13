# BTCP Security Model

Security is foundational to BTCP. This document describes the security architecture, threat model, and implementation requirements for BTCP-compliant systems.

## Security Principles

### 1. Least Privilege

Tools receive only the capabilities they explicitly require. No implicit permissions are granted.

### 2. User Consent

Users must explicitly approve capability grants. No silent or automatic permission escalation.

### 3. Defense in Depth

Multiple security layers protect against various attack vectors:
- Capability declarations
- User consent flows
- Sandboxed execution
- Input validation
- Output sanitization

### 4. Transparency

Users can inspect exactly what capabilities tools request and how they're used.

## Capability System

### Capability Structure

Capabilities follow a hierarchical namespace pattern:

```
category:resource:action
```

Examples:
- `dom:read` - Read DOM content
- `storage:local:write` - Write to localStorage
- `network:fetch:cross-origin` - Make cross-origin requests

### Capability Categories

#### DOM Capabilities

| Capability | Description | Risk Level |
|------------|-------------|------------|
| `dom:read` | Read page content and structure | Low |
| `dom:write` | Modify page content and structure | Medium |
| `dom:observe` | Observe DOM mutations | Low |
| `dom:shadow` | Access shadow DOM | Medium |

#### Storage Capabilities

| Capability | Description | Risk Level |
|------------|-------------|------------|
| `storage:local:read` | Read localStorage | Low |
| `storage:local:write` | Write localStorage | Medium |
| `storage:session:read` | Read sessionStorage | Low |
| `storage:session:write` | Write sessionStorage | Medium |
| `storage:indexed:read` | Read IndexedDB | Medium |
| `storage:indexed:write` | Write IndexedDB | High |
| `storage:cookie:read` | Read cookies | High |
| `storage:cookie:write` | Write cookies | Critical |

#### Network Capabilities

| Capability | Description | Risk Level |
|------------|-------------|------------|
| `network:fetch:same-origin` | Same-origin HTTP requests | Medium |
| `network:fetch:cross-origin` | Cross-origin HTTP requests | High |
| `network:websocket:same-origin` | Same-origin WebSocket | Medium |
| `network:websocket:cross-origin` | Cross-origin WebSocket | High |

#### Browser Capabilities

| Capability | Description | Risk Level |
|------------|-------------|------------|
| `clipboard:read` | Read clipboard content | High |
| `clipboard:write` | Write to clipboard | Medium |
| `media:camera` | Access camera | Critical |
| `media:microphone` | Access microphone | Critical |
| `geolocation` | Access location | High |
| `notifications` | Show notifications | Low |

### Capability Grants

#### Grant Types

1. **Session Grant**: Valid for current browser session
2. **Persistent Grant**: Remembered across sessions (requires user opt-in)
3. **One-Time Grant**: Valid for single tool invocation

#### Grant Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Agent     │     │ BTCP Client │     │    User     │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                    │
       │ requestCapabilities([...])             │
       │──────────────────►│                    │
       │                   │                    │
       │                   │ Show consent UI    │
       │                   │───────────────────►│
       │                   │                    │
       │                   │     User decision  │
       │                   │◄───────────────────│
       │                   │                    │
       │ { granted: [...] }│                    │
       │◄──────────────────│                    │
       │                   │                    │
```

#### Consent UI Requirements

BTCP-compliant clients MUST display consent dialogs that:

1. **Clearly identify the requesting agent**
2. **List all requested capabilities with plain-language descriptions**
3. **Show capability risk levels**
4. **Allow granular approval/denial per capability**
5. **Provide "remember this decision" option**
6. **Be non-dismissable without explicit choice**

Example consent dialog:

```
┌─────────────────────────────────────────────────────────────┐
│  "AI Assistant" is requesting access to:                     │
│                                                              │
│  ☑ Read page content                              [Low Risk] │
│    The AI will be able to read text and elements on         │
│    this page.                                                │
│                                                              │
│  ☐ Modify page content                         [Medium Risk] │
│    The AI will be able to change text, add or remove        │
│    elements on this page.                                    │
│                                                              │
│  ☐ Access clipboard                              [High Risk] │
│    The AI will be able to read data you've copied.          │
│                                                              │
│  ☐ Remember my choice for this site                         │
│                                                              │
│           [Deny All]  [Allow Selected]                       │
└─────────────────────────────────────────────────────────────┘
```

## Sandboxed Execution

### Sandbox Requirements

All BTCP tool execution MUST occur within a sandboxed environment that:

1. **Isolates tool code** from the main page context
2. **Restricts access** to only granted capabilities
3. **Prevents access** to global objects not related to granted capabilities
4. **Blocks unauthorized** network requests
5. **Terminates** on timeout or resource exhaustion

### Sandbox Implementations

#### Web Workers

Execution in a separate thread with message-passing:

```javascript
// Tool runs in Worker context
self.onmessage = async (event) => {
  const { toolName, args, capabilities } = event.data;

  // Capability-gated API access
  const api = createCapabilityGatedAPI(capabilities);

  const result = await executeTool(toolName, args, api);

  self.postMessage({ result });
};
```

**Isolation Properties:**
- Separate thread (no shared memory by default)
- No DOM access (must be proxied through capabilities)
- Controlled message passing

#### iframe Sandbox

Execution in a sandboxed iframe:

```html
<iframe
  sandbox="allow-scripts"
  src="about:blank"
  csp="default-src 'none'; script-src 'self'"
></iframe>
```

**Isolation Properties:**
- Separate browsing context
- Configurable sandbox flags
- Content Security Policy enforcement

#### SES Compartments

Secure ECMAScript compartments with frozen intrinsics:

```javascript
import { Compartment } from 'ses';

const compartment = new Compartment({
  globals: {
    // Only expose capability-gated APIs
    btcp: createCapabilityGatedAPI(capabilities)
  },
  __options__: {
    // Freeze all intrinsics
    __shimTransformsSources__: true
  }
});

const result = compartment.evaluate(toolCode);
```

**Isolation Properties:**
- Frozen JavaScript intrinsics
- Controlled global object
- No prototype pollution

#### WebAssembly Isolates

Hardware-isolated execution via WebAssembly:

```javascript
const wasmInstance = await WebAssembly.instantiate(toolModule, {
  btcp: {
    // Capability-gated imports
    readDOM: capabilities.includes('dom:read') ? readDOMImpl : blocked,
    writeDOM: capabilities.includes('dom:write') ? writeDOMImpl : blocked
  }
});

const result = wasmInstance.exports.execute(args);
```

**Isolation Properties:**
- Memory isolation at hardware level
- Explicit import/export interface
- Deterministic execution

### Sandbox Selection

| Requirement | Web Worker | iframe | SES | WebAssembly |
|-------------|------------|--------|-----|-------------|
| DOM Access | Proxied | Limited | Proxied | Proxied |
| Performance | Good | Good | Excellent | Excellent |
| Isolation Level | Medium | Medium | High | Very High |
| Browser Support | Excellent | Excellent | Good | Excellent |
| Startup Time | Fast | Medium | Fast | Slow |

## Input Validation

### Schema Validation

All tool inputs MUST be validated against declared JSON Schemas before execution:

```javascript
import Ajv from 'ajv';
import addFormats from 'ajv-formats';

const ajv = new Ajv({ allErrors: true });
addFormats(ajv);

function validateInput(tool, args) {
  const validate = ajv.compile(tool.inputSchema);

  if (!validate(args)) {
    throw new BTCPValidationError(
      'Invalid input parameters',
      validate.errors
    );
  }
}
```

### Sanitization

Inputs that will be used in sensitive contexts MUST be sanitized:

```javascript
// DOM insertion - sanitize HTML
import DOMPurify from 'dompurify';

function safeInnerHTML(element, html) {
  element.innerHTML = DOMPurify.sanitize(html);
}

// URL construction - validate and sanitize
function safeURL(urlString) {
  const url = new URL(urlString);

  // Only allow safe protocols
  if (!['http:', 'https:'].includes(url.protocol)) {
    throw new Error('Invalid URL protocol');
  }

  return url.href;
}
```

## Output Handling

### Sensitive Data Protection

Tools MUST NOT return sensitive data unless explicitly required:

```javascript
// Bad: Returns full cookie
function getCookies() {
  return document.cookie;  // May contain session tokens
}

// Good: Returns only requested cookie name existence
function hasCookie(name) {
  return document.cookie.split(';').some(c =>
    c.trim().startsWith(name + '=')
  );
}
```

### Output Size Limits

BTCP clients MUST enforce output size limits:

```javascript
const MAX_OUTPUT_SIZE = 1024 * 1024; // 1MB

function validateOutput(result) {
  const size = JSON.stringify(result).length;

  if (size > MAX_OUTPUT_SIZE) {
    throw new BTCPError('OUTPUT_TOO_LARGE', {
      message: `Output size ${size} exceeds limit ${MAX_OUTPUT_SIZE}`,
      size,
      limit: MAX_OUTPUT_SIZE
    });
  }

  return result;
}
```

## Transport Security

### Encryption Requirements

All BTCP communication MUST use encryption:

- **WebSocket**: WSS (TLS) only, except localhost for development
- **Extension Bridge**: Uses browser's secure messaging
- **Cloud Relay**: TLS with certificate pinning

### Authentication

Agent authentication options:

1. **API Key**: For cloud/server agents
2. **Extension Signature**: For browser extension agents
3. **Origin Verification**: For same-origin web agents

```javascript
// API Key authentication
const agent = new BTCPAgent({
  transport: 'cloud',
  apiKey: process.env.BTCP_API_KEY,
  // Key is sent via Authorization header
});

// Extension verification
const agent = new BTCPAgent({
  transport: 'extension',
  extensionId: 'verified-extension-id',
  // Browser verifies extension signature
});
```

## Threat Model

### Threats and Mitigations

| Threat | Description | Mitigation |
|--------|-------------|------------|
| Malicious Agent | Agent attempts unauthorized access | Capability system, user consent |
| Tool Escape | Tool breaks out of sandbox | Defense in depth, multiple sandbox layers |
| Data Exfiltration | Tool leaks sensitive data | Network capability restrictions, output validation |
| DoS | Tool consumes excessive resources | Timeouts, resource limits |
| Injection | Malicious input causes unintended execution | Schema validation, input sanitization |
| MITM | Attacker intercepts communication | TLS encryption, certificate validation |

### Security Boundaries

```
┌─────────────────────────────────────────────────────────────────┐
│                          Browser                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    Page Context                            │  │
│  │                                                            │  │
│  │  ┌──────────────────┐  ┌────────────────────────────────┐ │  │
│  │  │   Web App        │  │    BTCP Client                 │ │  │
│  │  │                  │  │                                │ │  │
│  │  │                  │  │  ┌──────────────────────────┐  │ │  │
│  │  │                  │◄─┼──┤ Capability Gate          │  │ │  │
│  │  │                  │  │  └───────────┬──────────────┘  │ │  │
│  │  └──────────────────┘  │              │                 │ │  │
│  │                        │  ┌───────────▼──────────────┐  │ │  │
│  │                        │  │      Sandbox             │  │ │  │
│  │                        │  │  ┌────────────────────┐  │  │ │  │
│  │                        │  │  │   Tool Instance    │  │  │ │  │
│  │                        │  │  └────────────────────┘  │  │ │  │
│  │                        │  └──────────────────────────┘  │ │  │
│  │                        │                                │ │  │
│  │                        └────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                    │                             │
│                              TLS   │                             │
└────────────────────────────────────┼─────────────────────────────┘
                                     │
                            ┌────────▼────────┐
                            │    AI Agent     │
                            └─────────────────┘
```

**Security Boundaries:**
1. Browser sandbox (OS-level)
2. Page context (same-origin policy)
3. BTCP sandbox (capability enforcement)
4. TLS encryption (transport)

## Compliance

### Security Audit Requirements

BTCP implementations SHOULD undergo security audits covering:

1. Sandbox escape testing
2. Capability bypass testing
3. Input fuzzing
4. Transport security verification
5. Consent flow verification

### Vulnerability Disclosure

Report security vulnerabilities to: security@btcp.dev

Provide:
- Detailed description
- Reproduction steps
- Impact assessment
- Suggested mitigation (if any)

## Best Practices

### For Tool Providers

1. **Declare minimal capabilities** - Only request what you need
2. **Validate all inputs** - Never trust agent-provided data
3. **Sanitize outputs** - Don't leak sensitive information
4. **Handle errors securely** - Don't expose internal details
5. **Test in sandbox** - Ensure tools work with restrictions

### For Tool Callers

1. **Verify manifests** - Check provider identity
2. **Request incrementally** - Ask for capabilities as needed
3. **Handle denials gracefully** - Provide alternative flows
4. **Audit tool calls** - Log capability usage
5. **Rotate credentials** - Refresh API keys regularly

### For BTCP Client Implementers

1. **Defense in depth** - Use multiple isolation layers
2. **Fail secure** - Default to denial on errors
3. **Audit logging** - Record all capability grants and usage
4. **Regular updates** - Keep sandbox implementations current
5. **Penetration testing** - Regularly test security boundaries
