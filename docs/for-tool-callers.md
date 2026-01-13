# BTCP for Tool Callers

This guide explains how to integrate BTCP into AI agents and applications to discover and invoke browser-based tools.

## Overview

As a tool caller, you connect to a BTCP-enabled browser to discover available tools and execute them. BTCP handles capability negotiation, sandboxed execution, and result delivery.

## Quick Start

### 1. Connect to BTCP Client

Establish a connection to the browser's BTCP client:

```javascript
import { BTCPAgent } from '@btcp/agent-sdk';

const agent = new BTCPAgent({
  // Connection via WebSocket
  transport: 'websocket',
  endpoint: 'ws://localhost:8765/btcp',

  // Or via browser extension bridge
  // transport: 'extension',
  // extensionId: 'your-extension-id'
});

await agent.connect();
```

### 2. Discover Available Tools

Query for registered tools and their schemas:

```javascript
// List all available tools
const tools = await agent.listTools();

console.log(tools);
// [
//   {
//     name: 'getPageContent',
//     description: 'Extracts the main content from the current page',
//     inputSchema: { ... },
//     capabilities: ['dom:read']
//   },
//   ...
// ]
```

### 3. Request Capabilities

Before calling tools, request the necessary capabilities:

```javascript
const granted = await agent.requestCapabilities([
  'dom:read',
  'storage:local:read'
]);

if (!granted) {
  console.log('User denied capability request');
}
```

### 4. Call Tools

Invoke tools with typed parameters:

```javascript
const result = await agent.callTool('getPageContent', {
  selector: 'article',
  format: 'markdown'
});

console.log(result);
// {
//   content: '# Article Title\n\nArticle content...',
//   wordCount: 523
// }
```

## Connection Methods

### WebSocket Transport

Direct WebSocket connection to a local BTCP server:

```javascript
const agent = new BTCPAgent({
  transport: 'websocket',
  endpoint: 'ws://localhost:8765/btcp',
  reconnect: true,
  reconnectInterval: 1000,
  maxReconnectAttempts: 5
});
```

### Browser Extension Bridge

Connect through a BTCP browser extension:

```javascript
const agent = new BTCPAgent({
  transport: 'extension',
  extensionId: 'btcp-bridge-extension-id',
  timeout: 30000
});
```

### Cloud Relay

Connect through BTCP Cloud for remote browser access:

```javascript
const agent = new BTCPAgent({
  transport: 'cloud',
  apiKey: process.env.BTCP_API_KEY,
  sessionId: 'user-session-123'
});
```

## Tool Discovery

### Listing Tools

Get all available tools:

```javascript
const tools = await agent.listTools();
```

### Filtering Tools

Filter by capability requirements:

```javascript
const readOnlyTools = await agent.listTools({
  capabilities: ['dom:read'],
  excludeCapabilities: ['dom:write']
});
```

### Getting Tool Details

Get complete information about a specific tool:

```javascript
const tool = await agent.getTool('getPageContent');

console.log(tool);
// {
//   name: 'getPageContent',
//   description: 'Extracts the main content...',
//   inputSchema: {
//     type: 'object',
//     properties: {
//       selector: { type: 'string', ... },
//       format: { type: 'string', enum: ['text', 'html', 'markdown'] }
//     }
//   },
//   outputSchema: { ... },
//   capabilities: ['dom:read'],
//   examples: [ ... ]
// }
```

### Manifest Metadata

Access manifest-level information:

```javascript
const manifests = await agent.listManifests();

for (const manifest of manifests) {
  console.log(`${manifest.name} v${manifest.version}`);
  console.log(`  Provider: ${manifest.provider?.name}`);
  console.log(`  Tools: ${manifest.tools.length}`);
}
```

## Capability Management

### Checking Current Capabilities

Query which capabilities are currently granted:

```javascript
const capabilities = await agent.getGrantedCapabilities();
// ['dom:read', 'storage:local:read']
```

### Requesting Capabilities

Request additional capabilities (prompts user):

```javascript
const result = await agent.requestCapabilities([
  'dom:write',
  'network:fetch:same-origin'
]);

if (result.granted) {
  console.log('All capabilities granted');
} else {
  console.log('Denied:', result.denied);
  console.log('Granted:', result.approved);
}
```

### Capability Preflight

Check if capabilities would be granted without prompting:

```javascript
const wouldGrant = await agent.preflightCapabilities([
  'dom:read',
  'clipboard:write'
]);

// {
//   'dom:read': 'granted',
//   'clipboard:write': 'prompt'  // Would require user approval
// }
```

## Calling Tools

### Basic Invocation

```javascript
const result = await agent.callTool('toolName', {
  param1: 'value1',
  param2: 123
});
```

### With Timeout

```javascript
const result = await agent.callTool('longRunningTool', {
  data: largeDataset
}, {
  timeout: 60000  // 60 seconds
});
```

### Batch Calls

Execute multiple tools in a single request:

```javascript
const results = await agent.callTools([
  { name: 'getPageTitle', arguments: {} },
  { name: 'getPageContent', arguments: { format: 'text' } },
  { name: 'getPageLinks', arguments: { external: true } }
]);

// Results in same order as requests
const [title, content, links] = results;
```

### Streaming Results

For tools that support streaming:

```javascript
const stream = await agent.callToolStreaming('analyzeDocument', {
  documentId: 'doc-123'
});

for await (const chunk of stream) {
  console.log('Progress:', chunk.progress);
  console.log('Partial result:', chunk.data);
}

const finalResult = stream.result;
```

## Error Handling

### Error Types

```javascript
import {
  BTCPConnectionError,
  BTCPCapabilityError,
  BTCPValidationError,
  BTCPExecutionError,
  BTCPTimeoutError
} from '@btcp/agent-sdk';

try {
  const result = await agent.callTool('riskyTool', params);
} catch (error) {
  if (error instanceof BTCPConnectionError) {
    // Connection lost - attempt reconnect
    await agent.reconnect();
  } else if (error instanceof BTCPCapabilityError) {
    // Missing required capability
    console.log('Need capability:', error.requiredCapability);
    await agent.requestCapabilities([error.requiredCapability]);
  } else if (error instanceof BTCPValidationError) {
    // Invalid parameters
    console.log('Validation errors:', error.validationErrors);
  } else if (error instanceof BTCPExecutionError) {
    // Tool threw an error
    console.log('Tool error:', error.code, error.message);
    console.log('Suggestion:', error.suggestion);
  } else if (error instanceof BTCPTimeoutError) {
    // Tool took too long
    console.log('Tool timed out after', error.timeout, 'ms');
  }
}
```

### Retry Logic

```javascript
async function callWithRetry(agent, toolName, args, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await agent.callTool(toolName, args);
    } catch (error) {
      if (error instanceof BTCPConnectionError && attempt < maxRetries) {
        await agent.reconnect();
        continue;
      }
      throw error;
    }
  }
}
```

## Integration Patterns

### LLM Function Calling

Convert BTCP tools to LLM function schemas:

```javascript
const tools = await agent.listTools();

// Convert to OpenAI function format
const functions = tools.map(tool => ({
  type: 'function',
  function: {
    name: tool.name,
    description: tool.description,
    parameters: tool.inputSchema
  }
}));

// Use with OpenAI
const response = await openai.chat.completions.create({
  model: 'gpt-4',
  messages: [...],
  tools: functions
});

// Execute function calls
for (const call of response.choices[0].message.tool_calls) {
  const result = await agent.callTool(
    call.function.name,
    JSON.parse(call.function.arguments)
  );
}
```

### Anthropic Claude Integration

```javascript
const tools = await agent.listTools();

// Convert to Claude tool format
const claudeTools = tools.map(tool => ({
  name: tool.name,
  description: tool.description,
  input_schema: tool.inputSchema
}));

// Use with Claude
const response = await anthropic.messages.create({
  model: 'claude-3-opus-20240229',
  max_tokens: 1024,
  tools: claudeTools,
  messages: [...]
});

// Execute tool use
for (const block of response.content) {
  if (block.type === 'tool_use') {
    const result = await agent.callTool(block.name, block.input);
  }
}
```

### MCP Bridge

Use BTCP tools alongside MCP:

```javascript
import { MCPClient } from '@modelcontextprotocol/sdk';
import { BTCPAgent } from '@btcp/agent-sdk';

const mcp = new MCPClient({ ... });
const btcp = new BTCPAgent({ ... });

// Combine tools from both protocols
const allTools = [
  ...await mcp.listTools(),
  ...await btcp.listTools()
];

// Route calls to appropriate protocol
async function callTool(name, args) {
  const btcpTools = await btcp.listTools();
  if (btcpTools.find(t => t.name === name)) {
    return btcp.callTool(name, args);
  }
  return mcp.callTool(name, args);
}
```

## Session Management

### Maintaining State

```javascript
// Get session info
const session = await agent.getSession();
console.log('Session ID:', session.id);
console.log('Connected at:', session.connectedAt);
console.log('Page URL:', session.pageUrl);

// Handle page navigation
agent.on('navigation', async (event) => {
  console.log('Page changed:', event.url);
  // Re-discover tools after navigation
  const tools = await agent.listTools();
});
```

### Graceful Disconnection

```javascript
// Disconnect cleanly
await agent.disconnect();

// Handle unexpected disconnection
agent.on('disconnect', async (event) => {
  if (event.reason === 'user_closed_page') {
    console.log('User closed the browser tab');
  } else {
    console.log('Connection lost, attempting reconnect...');
    await agent.reconnect();
  }
});
```

## Best Practices

### 1. Request Minimal Capabilities

Only request capabilities you need, and request them just-in-time:

```javascript
// Good: Request only what's needed
const readTools = tools.filter(t =>
  t.capabilities.every(c => c.endsWith(':read'))
);
await agent.requestCapabilities(['dom:read']);

// Avoid: Requesting everything upfront
await agent.requestCapabilities(['dom:read', 'dom:write', 'storage:*', ...]);
```

### 2. Validate Before Calling

Pre-validate parameters to get better error messages:

```javascript
import Ajv from 'ajv';
const ajv = new Ajv();

const tool = await agent.getTool('createPost');
const validate = ajv.compile(tool.inputSchema);

if (!validate(params)) {
  console.log('Invalid params:', validate.errors);
  return;
}

await agent.callTool('createPost', params);
```

### 3. Handle Tool Changes

Tools may change between calls (page navigation, dynamic registration):

```javascript
agent.on('tools:changed', async () => {
  // Refresh tool list
  const tools = await agent.listTools();
  updateAvailableTools(tools);
});
```

### 4. Use Timeouts Appropriately

Set timeouts based on expected tool duration:

```javascript
// Quick DOM read
await agent.callTool('getTitle', {}, { timeout: 5000 });

// File processing
await agent.callTool('processDocument', { ... }, { timeout: 120000 });
```

## TypeScript Support

Full type safety with generated types:

```typescript
import { BTCPAgent, ToolResult } from '@btcp/agent-sdk';
import type { GetPageContentInput, GetPageContentOutput } from './generated-types';

const agent = new BTCPAgent({ ... });

const result: ToolResult<GetPageContentOutput> = await agent.callTool<
  GetPageContentInput,
  GetPageContentOutput
>('getPageContent', {
  selector: 'main',
  format: 'markdown'
});

// result.content is typed as string
// result.wordCount is typed as number
```

## Next Steps

- **[API Reference: Agent SDK](./api/plugins/agent-sdk.md)**: Complete SDK documentation
- **[Protocols](./protocols/index.md)**: Transport layer details
- **[Security Model](./security.md)**: Understanding capability grants
- **[Implementation Guide](./implementation.md)**: Production best practices
