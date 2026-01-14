# BTCP Server

The BTCP Server is a transport relay that connects browser clients to AI agents. It provides MCP compatibility, enabling any MCP-capable AI agent to interact with browser-based tools.

## Overview

The BTCP Server acts as a **message broker** between browser clients and AI agents:

- **Does NOT execute tools** — tools run exclusively in the browser
- **Does NOT process data** — simply relays messages between endpoints
- **Provides MCP interface** — AI agents connect using standard MCP protocol
- **Manages sessions** — tracks which clients are connected to which agents

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│              │         │              │         │              │
│   Browser    │◄───────►│    BTCP      │◄───────►│   AI Agent   │
│   Client     │  BTCP   │    Server    │   MCP   │              │
│              │         │              │         │              │
└──────────────┘         └──────────────┘         └──────────────┘
     Tools                   Relay                   Consumer
     Execute                 Only
```

## Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                          BTCP Server                                 │
│                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │
│  │  Client Manager │  │ Session Manager │  │   Agent Manager     │  │
│  │                 │  │                 │  │                     │  │
│  │  - WebSocket    │  │  - Session IDs  │  │  - MCP Transport    │  │
│  │  - Auth         │  │  - Routing      │  │  - Tool Registry    │  │
│  │  - Heartbeat    │  │  - TTL          │  │  - Capability Sync  │  │
│  └────────┬────────┘  └────────┬────────┘  └──────────┬──────────┘  │
│           │                    │                      │             │
│           └────────────────────┼──────────────────────┘             │
│                                │                                     │
│                     ┌──────────▼──────────┐                         │
│                     │   Message Router    │                         │
│                     │                     │                         │
│                     │  Client ◄──► Agent  │                         │
│                     └─────────────────────┘                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Message Flow

```
AI Agent                    BTCP Server                 Browser Client
    │                            │                            │
    │  tools/list (MCP)          │                            │
    │───────────────────────────►│                            │
    │                            │  tools/list (BTCP)         │
    │                            │───────────────────────────►│
    │                            │                            │
    │                            │  [tool schemas]            │
    │                            │◄───────────────────────────│
    │  [tool schemas]            │                            │
    │◄───────────────────────────│                            │
    │                            │                            │
    │  tools/call (MCP)          │                            │
    │───────────────────────────►│                            │
    │                            │  tools/call (BTCP)         │
    │                            │───────────────────────────►│
    │                            │                            │
    │                            │     [executes in browser]  │
    │                            │                            │
    │                            │  [result]                  │
    │                            │◄───────────────────────────│
    │  [result]                  │                            │
    │◄───────────────────────────│                            │
    │                            │                            │
```

## Key Principles

### 1. Pure Transport Layer

The server NEVER:
- Inspects or modifies tool arguments
- Processes or transforms results
- Caches sensitive data
- Executes any tool logic

The server ONLY:
- Authenticates connections
- Routes messages between endpoints
- Manages session lifecycle
- Translates between BTCP and MCP protocols

### 2. MCP Compatibility

AI agents connect using standard MCP:

```javascript
// From AI agent perspective, it's just MCP
import { Client } from '@modelcontextprotocol/sdk/client';

const client = new Client({
  name: 'my-agent',
  version: '1.0.0'
});

// Connect to BTCP server via MCP
await client.connect(new StdioClientTransport({
  command: 'btcp-server',
  args: ['--session', sessionId]
}));

// Use tools exactly like MCP
const tools = await client.listTools();
const result = await client.callTool('getPageContent', { selector: 'main' });
```

### 3. Session-Based Routing

Each browser client creates a session that agents can join:

```
Session "abc-123"
├── Browser Client (connected via WebSocket)
└── AI Agent (connected via MCP)

Session "xyz-789"
├── Browser Client (connected via WebSocket)
├── AI Agent 1 (connected via MCP)
└── AI Agent 2 (connected via MCP)
```

## Server Modes

### Standalone Server

Runs as an independent process:

```bash
btcp-server --port 8765 --mcp-port 8766
```

- Browser clients connect via WebSocket on port 8765
- AI agents connect via MCP on port 8766

### Embedded Server

Runs within an existing application:

```javascript
import { BTCPServer } from '@btcp/server';

const server = new BTCPServer({
  clientPort: 8765,
  mcpPort: 8766
});

await server.start();
```

### Cloud Hosted

Managed BTCP server with additional features:

- Multi-tenant isolation
- Authentication/authorization
- Usage analytics
- Global distribution

## Configuration

### Basic Configuration

```yaml
# btcp-server.yaml
server:
  host: 0.0.0.0
  clientPort: 8765      # Browser clients connect here
  mcpPort: 8766         # AI agents connect here (MCP)

sessions:
  ttl: 3600             # Session timeout in seconds
  maxPerClient: 5       # Max agents per client session

security:
  requireAuth: true
  allowedOrigins:
    - https://example.com

logging:
  level: info
  auditLog: true
```

### Environment Variables

```bash
BTCP_HOST=0.0.0.0
BTCP_CLIENT_PORT=8765
BTCP_MCP_PORT=8766
BTCP_SESSION_TTL=3600
BTCP_REQUIRE_AUTH=true
BTCP_LOG_LEVEL=info
```

## Quick Start

### 1. Start the Server

```bash
# Using npx
npx @btcp/server

# Or install globally
npm install -g @btcp/server
btcp-server
```

### 2. Connect Browser Client

```javascript
// In browser
import { BTCPClient } from '@btcp/client';

const client = new BTCPClient({
  serverUrl: 'ws://localhost:8765'
});

// Connect and get session ID
const session = await client.connect();
console.log('Session ID:', session.id);
// Share this session ID with the AI agent

// Register tools
await client.registerManifest(myManifest);
```

### 3. Connect AI Agent (MCP)

```javascript
// In AI agent
import { Client } from '@modelcontextprotocol/sdk/client';
import { WebSocketClientTransport } from '@btcp/mcp-transport';

const client = new Client({
  name: 'my-agent',
  version: '1.0.0'
});

// Connect to BTCP server
const transport = new WebSocketClientTransport({
  url: 'ws://localhost:8766',
  sessionId: 'abc-123'  // Session ID from browser
});

await client.connect(transport);

// Now use tools - they execute in the browser!
const tools = await client.listTools();
const result = await client.callTool('getPageContent', {
  selector: 'main'
});
```

## Protocol Translation

The server translates between BTCP and MCP protocols:

### MCP Request → BTCP Request

```javascript
// MCP format (from agent)
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "getPageContent",
    "arguments": {
      "selector": "main"
    }
  }
}

// Translated to BTCP format (to client)
{
  "jsonrpc": "2.0",
  "id": "btcp-1",
  "method": "tools/call",
  "params": {
    "name": "getPageContent",
    "arguments": {
      "selector": "main"
    }
  }
}
```

### BTCP Response → MCP Response

```javascript
// BTCP format (from client)
{
  "jsonrpc": "2.0",
  "id": "btcp-1",
  "result": {
    "content": "Page content here...",
    "wordCount": 150
  }
}

// Translated to MCP format (to agent)
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"content\":\"Page content here...\",\"wordCount\":150}"
      }
    ]
  }
}
```

## Security

### Authentication

```javascript
// Client authentication
const client = new BTCPClient({
  serverUrl: 'ws://localhost:8765',
  auth: {
    type: 'bearer',
    token: 'client-token-here'
  }
});

// Agent authentication (via MCP)
const transport = new WebSocketClientTransport({
  url: 'ws://localhost:8766',
  sessionId: 'abc-123',
  auth: {
    type: 'bearer',
    token: 'agent-token-here'
  }
});
```

### Session Isolation

- Each session is isolated from others
- Agents can only access tools in their joined session
- Capability grants are session-scoped

### Transport Security

- TLS required in production (WSS)
- Origin validation for browser clients
- Rate limiting per session

## Related

- [Server Implementation](./implementation.md) - Detailed implementation guide
- [MCP Bridge](./mcp-bridge.md) - MCP protocol translation
- [Session Management](./sessions.md) - Session lifecycle
- [Deployment](./deployment.md) - Production deployment guide
