# Session Management

Sessions connect browser clients with AI agents through the BTCP server. This document describes session lifecycle, routing, and management.

## Overview

A session represents a connection between:
- **One browser client** (the tool provider)
- **One or more AI agents** (the tool consumers)

```
                    Session "abc-123"
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ Browser  │    │  Agent   │    │  Agent   │
    │ Client   │    │    1     │    │    2     │
    └──────────┘    └──────────┘    └──────────┘
       Owner          Member          Member
```

## Session Lifecycle

### 1. Creation

A session is created when a browser client connects:

```javascript
// Browser client initiates session
const client = new BTCPClient({
  serverUrl: 'ws://localhost:8765'
});

const session = await client.connect();
// session.id = "abc-123-xyz"
```

**Server-side:**
```javascript
// Session created on server
{
  id: "abc-123-xyz",
  createdAt: 1704067200000,
  expiresAt: 1704070800000,  // TTL: 1 hour
  owner: {
    type: "client",
    connectionId: "conn-001"
  },
  members: [],
  tools: [],
  capabilities: {
    granted: [],
    pending: []
  }
}
```

### 2. Agent Joining

AI agents join an existing session using the session ID:

```javascript
// Agent joins session via MCP
const transport = new WebSocketClientTransport({
  url: 'ws://localhost:8766',
  sessionId: 'abc-123-xyz'  // Provided by user/browser
});

await client.connect(transport);
```

**Server-side:**
```javascript
// Session after agent joins
{
  id: "abc-123-xyz",
  owner: { type: "client", connectionId: "conn-001" },
  members: [
    { type: "agent", connectionId: "conn-002", name: "claude-agent" }
  ],
  // ...
}
```

### 3. Tool Registration

Browser client registers tools that become available to agents:

```javascript
// Client registers manifest
await client.registerManifest(myManifest);
```

**Server-side:**
```javascript
// Session with registered tools
{
  id: "abc-123-xyz",
  tools: [
    {
      name: "getPageContent",
      manifestName: "page-tools",
      capabilities: ["dom:read"]
    },
    {
      name: "clickElement",
      manifestName: "page-tools",
      capabilities: ["dom:read", "dom:write"]
    }
  ],
  // ...
}
```

### 4. Message Routing

Server routes messages between session members:

```
Agent sends tools/call
        │
        ▼
┌───────────────────┐
│  Session Router   │
│                   │
│  1. Find session  │
│  2. Find client   │
│  3. Forward msg   │
└───────────────────┘
        │
        ▼
Client receives tools/call
        │
        ▼
Client executes in sandbox
        │
        ▼
Client sends result
        │
        ▼
┌───────────────────┐
│  Session Router   │
│                   │
│  1. Find session  │
│  2. Find agent    │
│  3. Forward result│
└───────────────────┘
        │
        ▼
Agent receives result
```

### 5. Termination

Sessions end when:
- Client disconnects
- TTL expires
- Explicit termination request

```javascript
// Client disconnects
await client.disconnect();

// Or explicit termination
await client.terminateSession();
```

## Session States

| State | Description |
|-------|-------------|
| `created` | Session created, no tools registered |
| `active` | Client connected, tools available |
| `idle` | No agents connected for idle timeout period |
| `expired` | TTL exceeded |
| `terminated` | Explicitly ended |

```
created ──► active ──► idle ──► expired
               │         │
               │         └──► active (agent reconnects)
               │
               └──► terminated
```

## Session ID Format

Session IDs are opaque strings with the following recommended format:

```
{random-part}-{checksum}

Example: a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

Properties:
- Globally unique
- URL-safe characters
- Not guessable (sufficient entropy)
- Optionally includes checksum for validation

## Multi-Agent Sessions

Multiple agents can join the same session:

```
Session "abc-123"
├── Browser Client (owner)
├── Agent: "claude-assistant" (member)
├── Agent: "research-bot" (member)
└── Agent: "code-helper" (member)
```

### Agent Isolation

Each agent:
- Sees all registered tools
- Shares capability grants
- Has independent message queue
- Cannot see other agents' requests

### Concurrent Access

When multiple agents call tools simultaneously:

```javascript
// Server handles concurrent requests
class SessionRouter {
  async routeToolCall(sessionId, agentId, toolCall) {
    const session = this.sessions.get(sessionId);

    // Queue request
    const requestId = await session.client.send({
      method: 'tools/call',
      params: toolCall,
      metadata: { agentId, requestId: toolCall.id }
    });

    // Track which agent made the request
    session.pendingRequests.set(requestId, agentId);
  }

  async routeToolResult(sessionId, requestId, result) {
    const session = this.sessions.get(sessionId);
    const agentId = session.pendingRequests.get(requestId);

    // Route result back to correct agent
    await session.agents.get(agentId).send(result);
    session.pendingRequests.delete(requestId);
  }
}
```

## Session Configuration

### TTL (Time-To-Live)

```yaml
sessions:
  ttl: 3600          # Base TTL in seconds
  maxTtl: 86400      # Maximum TTL (24 hours)
  idleTimeout: 300   # Terminate if no activity
```

### Renewal

Sessions can be renewed before expiration:

```javascript
// Client renews session
await client.renewSession();

// Or with new TTL
await client.renewSession({ ttl: 7200 });
```

### Limits

```yaml
sessions:
  maxPerClient: 5       # Max concurrent sessions per client
  maxAgentsPerSession: 10  # Max agents per session
  maxToolsPerSession: 100  # Max registered tools
```

## Session Events

### Client Events

```javascript
client.on('session:created', ({ sessionId }) => {
  console.log('Session created:', sessionId);
});

client.on('session:agent_joined', ({ agentId, agentName }) => {
  console.log('Agent joined:', agentName);
});

client.on('session:agent_left', ({ agentId }) => {
  console.log('Agent left:', agentId);
});

client.on('session:expiring', ({ expiresIn }) => {
  console.log('Session expires in:', expiresIn);
});
```

### Server Events

```javascript
server.on('session:created', ({ sessionId, clientId }) => {
  audit.log('Session created', { sessionId, clientId });
});

server.on('session:terminated', ({ sessionId, reason }) => {
  audit.log('Session terminated', { sessionId, reason });
});
```

## Session Persistence

For high-availability deployments, sessions can be persisted:

### Redis Backend

```javascript
import { RedisSessionStore } from '@btcp/server';

const server = new BTCPServer({
  sessionStore: new RedisSessionStore({
    url: 'redis://localhost:6379',
    prefix: 'btcp:session:'
  })
});
```

### Session Recovery

After server restart:

```javascript
// Clients reconnect with session ID
const client = new BTCPClient({
  serverUrl: 'ws://localhost:8765',
  sessionId: 'abc-123-xyz'  // Existing session
});

await client.reconnect();  // Recovers session state
```

## Security Considerations

### Session ID Protection

- Treat session IDs as secrets
- Don't expose in URLs or logs
- Use secure channels to share with agents

### Session Isolation

- Sessions cannot access other sessions
- Tools from one session not visible to others
- Capability grants are session-scoped

### Client Authority

- Only the session owner (client) can:
  - Register/unregister tools
  - Grant/revoke capabilities
  - Terminate the session

- Agents can only:
  - List tools
  - Call tools (with granted capabilities)
  - Request capabilities (requires user approval)

## Debugging Sessions

### Session Info Command

```bash
btcp-server sessions list
btcp-server sessions info abc-123-xyz
btcp-server sessions terminate abc-123-xyz
```

### Session Dump

```json
{
  "id": "abc-123-xyz",
  "state": "active",
  "createdAt": "2024-01-01T00:00:00Z",
  "expiresAt": "2024-01-01T01:00:00Z",
  "owner": {
    "type": "client",
    "connectionId": "conn-001",
    "connectedAt": "2024-01-01T00:00:00Z",
    "userAgent": "BTCPClient/1.0"
  },
  "members": [
    {
      "type": "agent",
      "connectionId": "conn-002",
      "name": "claude-agent",
      "connectedAt": "2024-01-01T00:00:30Z"
    }
  ],
  "tools": [
    { "name": "getPageContent", "capabilities": ["dom:read"] }
  ],
  "capabilities": {
    "granted": ["dom:read"],
    "pending": ["dom:write"]
  },
  "stats": {
    "toolCalls": 42,
    "errors": 2,
    "lastActivity": "2024-01-01T00:30:00Z"
  }
}
```

## Related

- [Server Overview](./index.md)
- [MCP Bridge](./mcp-bridge.md)
- [Security Model](../security.md)
