# MCP Bridge

The MCP Bridge translates between the Model Context Protocol (MCP) and BTCP, enabling any MCP-compatible AI agent to use browser-based tools without modification.

## Overview

The bridge provides a standard MCP interface while routing tool calls to browser clients:

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│                 │      │   MCP Bridge    │      │                 │
│    AI Agent     │◄────►│                 │◄────►│  Browser Client │
│                 │ MCP  │  ┌───────────┐  │ BTCP │                 │
│  Claude, GPT,   │      │  │ Translate │  │      │  Tools execute  │
│  Custom Agent   │      │  └───────────┘  │      │  here           │
│                 │      │                 │      │                 │
└─────────────────┘      └─────────────────┘      └─────────────────┘
```

## MCP Protocol Support

### Supported MCP Methods

| MCP Method | Description | BTCP Equivalent |
|------------|-------------|-----------------|
| `initialize` | Initialize connection | Session join |
| `tools/list` | List available tools | `tools/list` |
| `tools/call` | Execute a tool | `tools/call` |
| `ping` | Health check | Heartbeat |

### Initialization

When an agent connects via MCP, the bridge:

1. Validates the session ID
2. Joins the agent to the session
3. Returns server capabilities

```javascript
// MCP Initialize Request
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {}
    },
    "clientInfo": {
      "name": "my-agent",
      "version": "1.0.0"
    }
  }
}

// MCP Initialize Response
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {
        "listChanged": true
      }
    },
    "serverInfo": {
      "name": "btcp-server",
      "version": "1.0.0"
    }
  }
}
```

### Tool Listing

The bridge aggregates tools from connected browser clients:

```javascript
// MCP tools/list Request
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list"
}

// MCP tools/list Response (translated from BTCP)
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "getPageContent",
        "description": "Extracts the main content from the current page",
        "inputSchema": {
          "type": "object",
          "properties": {
            "selector": {
              "type": "string",
              "description": "CSS selector for content area"
            }
          }
        }
      },
      {
        "name": "clickElement",
        "description": "Clicks an element on the page",
        "inputSchema": {
          "type": "object",
          "properties": {
            "selector": {
              "type": "string",
              "description": "CSS selector for element to click"
            }
          },
          "required": ["selector"]
        }
      }
    ]
  }
}
```

### Tool Calling

Tool calls are forwarded to the browser client:

```javascript
// MCP tools/call Request
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "getPageContent",
    "arguments": {
      "selector": "article"
    }
  }
}

// Bridge forwards to BTCP client...
// Client executes tool in browser sandbox...
// Bridge receives result and translates:

// MCP tools/call Response
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"content\":\"Article text here...\",\"wordCount\":523}"
      }
    ]
  }
}
```

## Protocol Translation

### Request Translation (MCP → BTCP)

```javascript
class MCPToBTCPTranslator {
  translateRequest(mcpRequest) {
    const { method, params, id } = mcpRequest;

    switch (method) {
      case 'tools/list':
        return {
          jsonrpc: '2.0',
          id: this.mapId(id),
          method: 'tools/list',
          params: {}
        };

      case 'tools/call':
        return {
          jsonrpc: '2.0',
          id: this.mapId(id),
          method: 'tools/call',
          params: {
            name: params.name,
            arguments: params.arguments
          }
        };

      default:
        throw new Error(`Unsupported method: ${method}`);
    }
  }
}
```

### Response Translation (BTCP → MCP)

```javascript
class BTCPToMCPTranslator {
  translateResponse(btcpResponse, method) {
    const { result, error, id } = btcpResponse;

    if (error) {
      return {
        jsonrpc: '2.0',
        id: this.unmapId(id),
        error: {
          code: this.mapErrorCode(error.code),
          message: error.message
        }
      };
    }

    switch (method) {
      case 'tools/list':
        return {
          jsonrpc: '2.0',
          id: this.unmapId(id),
          result: {
            tools: result.map(tool => ({
              name: tool.name,
              description: tool.description,
              inputSchema: tool.inputSchema
            }))
          }
        };

      case 'tools/call':
        return {
          jsonrpc: '2.0',
          id: this.unmapId(id),
          result: {
            content: [
              {
                type: 'text',
                text: JSON.stringify(result)
              }
            ]
          }
        };

      default:
        return {
          jsonrpc: '2.0',
          id: this.unmapId(id),
          result
        };
    }
  }
}
```

## Capability Mapping

### BTCP Capabilities → MCP

BTCP's capability system is translated to MCP tool availability:

```javascript
// BTCP tool with capabilities
{
  "name": "writeToStorage",
  "capabilities": ["storage:local:write"]
}

// If capability not granted, tool is excluded from MCP listing
// If capability granted, tool appears in MCP tools/list
```

### Capability Requests via MCP

Agents can request capabilities through a special meta-tool:

```javascript
// Request capability via MCP tool call
{
  "method": "tools/call",
  "params": {
    "name": "__btcp_request_capabilities",
    "arguments": {
      "capabilities": ["dom:write", "storage:local:read"]
    }
  }
}

// Response indicates grant status
{
  "result": {
    "content": [{
      "type": "text",
      "text": "{\"granted\":[\"storage:local:read\"],\"denied\":[\"dom:write\"],\"pending\":[]}"
    }]
  }
}
```

## Transport Options

### Stdio Transport (Claude Desktop)

For local AI applications like Claude Desktop:

```javascript
// btcp-mcp-server.js - Run as MCP server
import { Server } from '@modelcontextprotocol/sdk/server';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio';
import { BTCPBridge } from '@btcp/server';

const bridge = new BTCPBridge({
  btcpServerUrl: 'ws://localhost:8765',
  sessionId: process.env.BTCP_SESSION_ID
});

const server = new Server({
  name: 'btcp-bridge',
  version: '1.0.0'
}, {
  capabilities: { tools: {} }
});

// Register tool handlers
server.setRequestHandler('tools/list', async () => {
  return bridge.listTools();
});

server.setRequestHandler('tools/call', async (request) => {
  return bridge.callTool(request.params.name, request.params.arguments);
});

// Connect via stdio
const transport = new StdioServerTransport();
await server.connect(transport);
```

**Claude Desktop Configuration:**

```json
{
  "mcpServers": {
    "btcp": {
      "command": "node",
      "args": ["/path/to/btcp-mcp-server.js"],
      "env": {
        "BTCP_SESSION_ID": "your-session-id"
      }
    }
  }
}
```

### WebSocket Transport

For remote AI agents:

```javascript
import { WebSocketClientTransport } from '@btcp/mcp-transport';
import { Client } from '@modelcontextprotocol/sdk/client';

const transport = new WebSocketClientTransport({
  url: 'wss://btcp.example.com:8766/mcp',
  sessionId: 'abc-123'
});

const client = new Client({
  name: 'remote-agent',
  version: '1.0.0'
});

await client.connect(transport);
```

### HTTP+SSE Transport

For serverless environments:

```javascript
import { SSEClientTransport } from '@modelcontextprotocol/sdk/client/sse';

const transport = new SSEClientTransport(
  new URL(`https://btcp.example.com/mcp/sse?session=${sessionId}`)
);

const client = new Client({ name: 'serverless-agent', version: '1.0.0' });
await client.connect(transport);
```

## Tool Changed Notifications

When browser tools change (page navigation, dynamic registration), the bridge notifies agents:

```javascript
// MCP Notification (server → agent)
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

Agents should re-fetch the tool list when receiving this notification.

## Error Handling

### Error Code Mapping

| BTCP Error | MCP Error Code | Description |
|------------|----------------|-------------|
| `TOOL_NOT_FOUND` | -32601 | Method not found |
| `INVALID_PARAMS` | -32602 | Invalid params |
| `CAPABILITY_DENIED` | -32603 | Internal error |
| `EXECUTION_ERROR` | -32603 | Internal error |
| `TIMEOUT` | -32603 | Internal error |

### Error Response Format

```javascript
// BTCP error
{
  "error": {
    "code": "CAPABILITY_DENIED",
    "message": "Capability not granted: dom:write"
  }
}

// Translated to MCP error
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32603,
    "message": "Capability not granted: dom:write",
    "data": {
      "btcpCode": "CAPABILITY_DENIED"
    }
  }
}
```

## Implementation Example

### Complete Bridge Implementation

```javascript
import { Server } from '@modelcontextprotocol/sdk/server';
import { WebSocket } from 'ws';

class BTCPMCPBridge {
  constructor(options) {
    this.btcpUrl = options.btcpServerUrl;
    this.sessionId = options.sessionId;
    this.btcpSocket = null;
    this.pendingRequests = new Map();
  }

  async connect() {
    return new Promise((resolve, reject) => {
      this.btcpSocket = new WebSocket(this.btcpUrl);

      this.btcpSocket.on('open', () => {
        // Join session
        this.sendBTCP({
          method: 'session/join',
          params: { sessionId: this.sessionId }
        }).then(resolve).catch(reject);
      });

      this.btcpSocket.on('message', (data) => {
        this.handleBTCPMessage(JSON.parse(data));
      });

      this.btcpSocket.on('error', reject);
    });
  }

  async listTools() {
    const btcpTools = await this.sendBTCP({
      method: 'tools/list',
      params: {}
    });

    return {
      tools: btcpTools.map(tool => ({
        name: tool.name,
        description: tool.description,
        inputSchema: tool.inputSchema
      }))
    };
  }

  async callTool(name, args) {
    const result = await this.sendBTCP({
      method: 'tools/call',
      params: { name, arguments: args }
    });

    return {
      content: [{
        type: 'text',
        text: JSON.stringify(result)
      }]
    };
  }

  sendBTCP(request) {
    return new Promise((resolve, reject) => {
      const id = crypto.randomUUID();
      this.pendingRequests.set(id, { resolve, reject });

      this.btcpSocket.send(JSON.stringify({
        jsonrpc: '2.0',
        id,
        ...request
      }));
    });
  }

  handleBTCPMessage(message) {
    if (message.id && this.pendingRequests.has(message.id)) {
      const { resolve, reject } = this.pendingRequests.get(message.id);
      this.pendingRequests.delete(message.id);

      if (message.error) {
        reject(message.error);
      } else {
        resolve(message.result);
      }
    }
  }
}
```

## Related

- [Server Overview](./index.md)
- [Session Management](./sessions.md)
- [For Tool Callers](../for-tool-callers.md)
- [MCP Specification](https://modelcontextprotocol.io)
