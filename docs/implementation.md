# Implementation Guide

This guide provides best practices and recommendations for implementing BTCP-compliant clients, tools, and agent integrations.

## Overview

A complete BTCP implementation includes:

1. **BTCP Client**: Runs in browser, manages tools and sandboxes
2. **Transport Layer**: Communication with external agents
3. **Tool Runtime**: Sandboxed execution environment
4. **Agent SDK**: Library for AI agents to interact with BTCP

## BTCP Client Implementation

### Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                      BTCP Client                             │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │   Registry   │  │   Security   │  │    Transport     │  │
│  │              │  │              │  │                  │  │
│  │ - Manifests  │  │ - Caps mgmt  │  │ - WebSocket      │  │
│  │ - Tools      │  │ - Consent UI │  │ - Extension      │  │
│  │ - Schemas    │  │ - Validation │  │ - Cloud relay    │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │   Executor   │  │   Sandbox    │  │     Logger       │  │
│  │              │  │   Manager    │  │                  │  │
│  │ - Dispatch   │  │ - Worker     │  │ - Audit trail    │  │
│  │ - Timeout    │  │ - iframe     │  │ - Diagnostics    │  │
│  │ - Results    │  │ - SES/Wasm   │  │ - Metrics        │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Registry Implementation

```javascript
class ManifestRegistry {
  constructor() {
    this.manifests = new Map();
    this.tools = new Map();
  }

  async register(manifest, implementations) {
    // Validate manifest schema
    const validation = await this.validateManifest(manifest);
    if (!validation.valid) {
      throw new BTCPValidationError(validation.errors);
    }

    // Check for conflicts
    for (const tool of manifest.tools) {
      if (this.tools.has(tool.name)) {
        throw new BTCPError('TOOL_CONFLICT', {
          message: `Tool already registered: ${tool.name}`
        });
      }
    }

    // Register manifest and tools
    this.manifests.set(manifest.name, {
      manifest,
      implementations,
      registeredAt: Date.now()
    });

    for (const tool of manifest.tools) {
      this.tools.set(tool.name, {
        tool,
        manifestName: manifest.name,
        implementation: implementations[tool.name]
      });
    }

    this.emit('manifest:registered', { name: manifest.name });
  }

  async unregister(manifestName) {
    const entry = this.manifests.get(manifestName);
    if (!entry) return false;

    for (const tool of entry.manifest.tools) {
      this.tools.delete(tool.name);
    }

    this.manifests.delete(manifestName);
    this.emit('manifest:unregistered', { name: manifestName });

    return true;
  }

  listTools(filter = {}) {
    let tools = Array.from(this.tools.values()).map(e => e.tool);

    if (filter.capabilities) {
      tools = tools.filter(t =>
        filter.capabilities.every(c => t.capabilities.includes(c))
      );
    }

    if (filter.tags) {
      tools = tools.filter(t =>
        filter.tags.some(tag => t.tags?.includes(tag))
      );
    }

    return tools;
  }
}
```

### Security Manager

```javascript
class SecurityManager {
  constructor() {
    this.grants = new Map();
    this.consentUI = new ConsentUI();
  }

  async requestCapabilities(capabilities, options = {}) {
    const needed = capabilities.filter(c => !this.isGranted(c));

    if (needed.length === 0) {
      return { granted: true, capabilities };
    }

    // Check if capabilities can be auto-granted
    const autoGrantable = needed.filter(c => this.canAutoGrant(c));
    const requireConsent = needed.filter(c => !this.canAutoGrant(c));

    // Auto-grant low-risk capabilities
    for (const cap of autoGrantable) {
      this.grant(cap, { type: 'auto' });
    }

    // Request consent for remaining
    if (requireConsent.length > 0) {
      const result = await this.consentUI.show({
        capabilities: requireConsent,
        requester: options.requester
      });

      if (result.denied.length > 0) {
        return {
          granted: false,
          approved: result.approved,
          denied: result.denied
        };
      }

      for (const cap of result.approved) {
        this.grant(cap, {
          type: 'user',
          persistent: result.remember
        });
      }
    }

    return { granted: true, capabilities };
  }

  isGranted(capability) {
    return this.grants.has(capability);
  }

  canAutoGrant(capability) {
    // Only auto-grant lowest-risk capabilities
    const autoGrantList = ['notifications'];
    return autoGrantList.includes(capability);
  }

  grant(capability, metadata) {
    this.grants.set(capability, {
      ...metadata,
      grantedAt: Date.now()
    });

    this.emit('capability:granted', { capability, metadata });
  }

  revoke(capability) {
    this.grants.delete(capability);
    this.emit('capability:revoked', { capability });
  }

  validateToolCapabilities(tool) {
    const missing = tool.capabilities.filter(c => !this.isGranted(c));

    if (missing.length > 0) {
      throw new BTCPCapabilityError({
        message: 'Missing required capabilities',
        required: tool.capabilities,
        missing
      });
    }
  }
}
```

### Executor Implementation

```javascript
class ToolExecutor {
  constructor(registry, security, sandboxManager) {
    this.registry = registry;
    this.security = security;
    this.sandboxManager = sandboxManager;
  }

  async execute(toolName, args, options = {}) {
    const entry = this.registry.tools.get(toolName);

    if (!entry) {
      throw new BTCPError('TOOL_NOT_FOUND', {
        message: `Unknown tool: ${toolName}`
      });
    }

    const { tool, implementation, manifestName } = entry;

    // Validate capabilities
    this.security.validateToolCapabilities(tool);

    // Validate input
    const validation = this.validateInput(tool.inputSchema, args);
    if (!validation.valid) {
      throw new BTCPValidationError({
        message: 'Invalid input parameters',
        errors: validation.errors
      });
    }

    // Get timeout
    const timeout = options.timeout || tool.timeout || 30000;

    // Execute in sandbox
    const sandbox = await this.sandboxManager.getSandbox(
      tool.capabilities
    );

    try {
      const result = await Promise.race([
        sandbox.execute(implementation, args),
        this.timeoutPromise(timeout)
      ]);

      // Validate output if schema provided
      if (tool.outputSchema) {
        this.validateOutput(tool.outputSchema, result);
      }

      this.emit('tool:executed', {
        tool: toolName,
        duration: Date.now() - startTime,
        success: true
      });

      return result;

    } catch (error) {
      this.emit('tool:error', {
        tool: toolName,
        error: error.message
      });

      throw error;

    } finally {
      sandbox.cleanup();
    }
  }

  timeoutPromise(ms) {
    return new Promise((_, reject) => {
      setTimeout(() => {
        reject(new BTCPTimeoutError(`Execution timed out after ${ms}ms`));
      }, ms);
    });
  }
}
```

## Transport Implementation

### WebSocket Transport

```javascript
class WebSocketTransport {
  constructor(options = {}) {
    this.url = options.url || 'ws://localhost:8765/btcp';
    this.reconnect = options.reconnect ?? true;
    this.ws = null;
    this.pending = new Map();
  }

  async connect() {
    return new Promise((resolve, reject) => {
      this.ws = new WebSocket(this.url);

      this.ws.onopen = () => {
        this.emit('connected');
        resolve();
      };

      this.ws.onerror = (error) => {
        reject(new BTCPConnectionError(error.message));
      };

      this.ws.onmessage = (event) => {
        this.handleMessage(JSON.parse(event.data));
      };

      this.ws.onclose = () => {
        this.emit('disconnected');
        if (this.reconnect) {
          this.scheduleReconnect();
        }
      };
    });
  }

  async send(method, params) {
    const id = crypto.randomUUID();

    return new Promise((resolve, reject) => {
      this.pending.set(id, { resolve, reject });

      const message = {
        jsonrpc: '2.0',
        id,
        method,
        params
      };

      this.ws.send(JSON.stringify(message));
    });
  }

  handleMessage(message) {
    if (message.id && this.pending.has(message.id)) {
      const { resolve, reject } = this.pending.get(message.id);
      this.pending.delete(message.id);

      if (message.error) {
        reject(new BTCPError(message.error.code, message.error));
      } else {
        resolve(message.result);
      }
    } else if (message.method) {
      // Handle incoming requests (e.g., tool calls from agent)
      this.handleRequest(message);
    }
  }

  async handleRequest(message) {
    try {
      const result = await this.client.handleMethod(
        message.method,
        message.params
      );

      this.ws.send(JSON.stringify({
        jsonrpc: '2.0',
        id: message.id,
        result
      }));

    } catch (error) {
      this.ws.send(JSON.stringify({
        jsonrpc: '2.0',
        id: message.id,
        error: {
          code: error.code || -32603,
          message: error.message
        }
      }));
    }
  }
}
```

## Agent SDK Implementation

### Core Agent Class

```javascript
class BTCPAgent {
  constructor(options) {
    this.transport = this.createTransport(options);
    this.tools = new Map();
  }

  async connect() {
    await this.transport.connect();
    await this.refreshTools();
  }

  async refreshTools() {
    const tools = await this.transport.send('tools/list', {});
    this.tools.clear();
    for (const tool of tools) {
      this.tools.set(tool.name, tool);
    }
  }

  async listTools(filter = {}) {
    return this.transport.send('tools/list', filter);
  }

  async getTool(name) {
    return this.transport.send('tools/get', { name });
  }

  async requestCapabilities(capabilities) {
    return this.transport.send('capabilities/request', { capabilities });
  }

  async callTool(name, args, options = {}) {
    const tool = this.tools.get(name);

    if (!tool) {
      throw new BTCPError('TOOL_NOT_FOUND', {
        message: `Unknown tool: ${name}`
      });
    }

    return this.transport.send('tools/call', {
      name,
      arguments: args,
      timeout: options.timeout
    });
  }

  async callTools(calls) {
    return this.transport.send('tools/callBatch', { calls });
  }

  // Convert tools to LLM format
  toLLMTools(format = 'openai') {
    const tools = Array.from(this.tools.values());

    switch (format) {
      case 'openai':
        return tools.map(t => ({
          type: 'function',
          function: {
            name: t.name,
            description: t.description,
            parameters: t.inputSchema
          }
        }));

      case 'anthropic':
        return tools.map(t => ({
          name: t.name,
          description: t.description,
          input_schema: t.inputSchema
        }));

      default:
        throw new Error(`Unknown format: ${format}`);
    }
  }
}
```

## Testing

### Unit Testing Tools

```javascript
import { createMockBTCP } from '@btcp/testing';

describe('MyTool', () => {
  let btcp;

  beforeEach(() => {
    btcp = createMockBTCP({
      dom: {
        'h1': { textContent: 'Hello World' },
        '#content': { textContent: 'Page content here' }
      }
    });
  });

  it('extracts page title', async () => {
    const result = await myTool({ selector: 'h1' }, btcp);
    expect(result.text).toBe('Hello World');
  });

  it('handles missing elements', async () => {
    await expect(
      myTool({ selector: '#nonexistent' }, btcp)
    ).rejects.toThrow('ELEMENT_NOT_FOUND');
  });
});
```

### Integration Testing

```javascript
import { BTCPTestClient } from '@btcp/testing';

describe('BTCP Integration', () => {
  let client;

  beforeAll(async () => {
    client = new BTCPTestClient();
    await client.registerManifest(myManifest);
    await client.grantCapabilities(['dom:read']);
  });

  it('executes tool correctly', async () => {
    const result = await client.callTool('getPageTitle', {});
    expect(result).toHaveProperty('title');
  });

  it('rejects without capability', async () => {
    await client.revokeCapabilities(['dom:write']);

    await expect(
      client.callTool('setPageTitle', { title: 'New Title' })
    ).rejects.toThrow('CAPABILITY_DENIED');
  });
});
```

### E2E Testing with Playwright

```javascript
import { test, expect } from '@playwright/test';

test('BTCP tool execution', async ({ page }) => {
  // Navigate to page with BTCP client
  await page.goto('http://localhost:3000');

  // Wait for BTCP to initialize
  await page.waitForFunction(() => window.btcpReady);

  // Execute tool via page context
  const result = await page.evaluate(async () => {
    return await window.btcp.callTool('getPageContent', {
      selector: 'main'
    });
  });

  expect(result.content).toContain('Expected text');
});
```

## Performance Optimization

### Tool Caching

```javascript
class CachedExecutor {
  constructor(executor) {
    this.executor = executor;
    this.cache = new Map();
  }

  async execute(toolName, args, options = {}) {
    if (options.cache) {
      const key = this.cacheKey(toolName, args);
      const cached = this.cache.get(key);

      if (cached && Date.now() - cached.timestamp < options.cacheTTL) {
        return cached.result;
      }
    }

    const result = await this.executor.execute(toolName, args, options);

    if (options.cache) {
      const key = this.cacheKey(toolName, args);
      this.cache.set(key, {
        result,
        timestamp: Date.now()
      });
    }

    return result;
  }

  cacheKey(toolName, args) {
    return `${toolName}:${JSON.stringify(args)}`;
  }
}
```

### Sandbox Pooling

```javascript
class SandboxPool {
  constructor(options = {}) {
    this.maxSize = options.maxSize || 5;
    this.pool = [];
  }

  async acquire(capabilities) {
    // Try to reuse existing sandbox
    const existing = this.pool.find(s =>
      this.capabilitiesMatch(s.capabilities, capabilities)
    );

    if (existing) {
      this.pool = this.pool.filter(s => s !== existing);
      return existing;
    }

    // Create new sandbox
    return this.createSandbox(capabilities);
  }

  release(sandbox) {
    if (this.pool.length < this.maxSize) {
      this.pool.push(sandbox);
    } else {
      sandbox.terminate();
    }
  }
}
```

## Monitoring and Observability

### Metrics Collection

```javascript
class BTCPMetrics {
  constructor() {
    this.counters = new Map();
    this.histograms = new Map();
  }

  recordToolCall(toolName, duration, success) {
    this.increment(`tool.${toolName}.calls`);
    this.increment(`tool.${toolName}.${success ? 'success' : 'error'}`);
    this.recordHistogram(`tool.${toolName}.duration`, duration);
  }

  recordCapabilityRequest(capabilities, granted) {
    for (const cap of capabilities) {
      this.increment(`capability.${cap}.requested`);
      if (granted) {
        this.increment(`capability.${cap}.granted`);
      }
    }
  }

  getMetrics() {
    return {
      counters: Object.fromEntries(this.counters),
      histograms: Object.fromEntries(this.histograms)
    };
  }
}
```

### Audit Logging

```javascript
class AuditLogger {
  constructor(options = {}) {
    this.destination = options.destination || console;
  }

  logToolExecution(event) {
    this.log({
      type: 'TOOL_EXECUTION',
      timestamp: Date.now(),
      tool: event.tool,
      capabilities: event.capabilities,
      duration: event.duration,
      success: event.success,
      error: event.error?.message
    });
  }

  logCapabilityGrant(event) {
    this.log({
      type: 'CAPABILITY_GRANT',
      timestamp: Date.now(),
      capability: event.capability,
      grantType: event.type,
      persistent: event.persistent
    });
  }

  log(entry) {
    this.destination.log(JSON.stringify(entry));
  }
}
```

## Deployment Considerations

### Production Checklist

- [ ] Enable TLS for all transports
- [ ] Configure CSP headers
- [ ] Set up rate limiting
- [ ] Implement audit logging
- [ ] Configure timeouts appropriately
- [ ] Test sandbox security
- [ ] Set up monitoring/alerting
- [ ] Document tool capabilities
- [ ] Review capability grants
- [ ] Test error handling

### Security Hardening

```javascript
// Production BTCP client configuration
const client = new BTCPClient({
  // Require secure transport
  transport: {
    type: 'websocket',
    url: 'wss://btcp.example.com',
    requireTLS: true
  },

  // Strict sandbox configuration
  sandbox: {
    type: 'ses',
    timeout: 30000,
    memoryLimit: 50 * 1024 * 1024
  },

  // Security options
  security: {
    requireUserConsent: true,
    auditLogging: true,
    rateLimit: {
      requests: 100,
      window: 60000
    }
  }
});
```

## Related

- [For Tool Providers](./for-tool-providers.md)
- [For Tool Callers](./for-tool-callers.md)
- [Security Model](./security.md)
- [Protocols Overview](./protocols/index.md)
