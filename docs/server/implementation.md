# Server Implementation

This guide provides detailed implementation guidance for building a BTCP server.

## Architecture

### Core Components

```
┌────────────────────────────────────────────────────────────────────┐
│                           BTCP Server                               │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                     Connection Layer                          │  │
│  │  ┌────────────────┐              ┌────────────────────────┐  │  │
│  │  │ Client Handler │              │    MCP Handler         │  │  │
│  │  │ (WebSocket)    │              │    (Stdio/WS/SSE)      │  │  │
│  │  └───────┬────────┘              └───────────┬────────────┘  │  │
│  └──────────┼───────────────────────────────────┼───────────────┘  │
│             │                                   │                   │
│  ┌──────────▼───────────────────────────────────▼───────────────┐  │
│  │                     Session Layer                             │  │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  │  │
│  │  │ Session Store  │  │ Session Router │  │ Session Events │  │  │
│  │  └────────────────┘  └────────────────┘  └────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                     Protocol Layer                            │  │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  │  │
│  │  │ BTCP Protocol  │  │ MCP Translator │  │ Message Queue  │  │  │
│  │  └────────────────┘  └────────────────┘  └────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

## Reference Implementation

### Server Class

```typescript
import { WebSocketServer, WebSocket } from 'ws';
import { EventEmitter } from 'events';

interface ServerOptions {
  clientPort: number;
  mcpPort: number;
  sessionTTL?: number;
}

interface Session {
  id: string;
  client: WebSocket | null;
  agents: Map<string, WebSocket>;
  tools: Tool[];
  capabilities: Set<string>;
  createdAt: number;
  expiresAt: number;
}

class BTCPServer extends EventEmitter {
  private clientServer: WebSocketServer;
  private mcpServer: WebSocketServer;
  private sessions: Map<string, Session>;
  private connectionToSession: Map<WebSocket, string>;

  constructor(private options: ServerOptions) {
    super();
    this.sessions = new Map();
    this.connectionToSession = new Map();
  }

  async start(): Promise<void> {
    // Start client WebSocket server
    this.clientServer = new WebSocketServer({
      port: this.options.clientPort
    });

    this.clientServer.on('connection', (ws, req) => {
      this.handleClientConnection(ws, req);
    });

    // Start MCP WebSocket server
    this.mcpServer = new WebSocketServer({
      port: this.options.mcpPort
    });

    this.mcpServer.on('connection', (ws, req) => {
      this.handleAgentConnection(ws, req);
    });

    console.log(`BTCP Server started`);
    console.log(`  Clients: ws://localhost:${this.options.clientPort}`);
    console.log(`  Agents:  ws://localhost:${this.options.mcpPort}`);
  }

  private handleClientConnection(ws: WebSocket, req: any): void {
    console.log('Client connected');

    ws.on('message', async (data) => {
      try {
        const message = JSON.parse(data.toString());
        await this.handleClientMessage(ws, message);
      } catch (error) {
        this.sendError(ws, null, error);
      }
    });

    ws.on('close', () => {
      this.handleClientDisconnect(ws);
    });
  }

  private async handleClientMessage(ws: WebSocket, message: any): Promise<void> {
    const { method, params, id } = message;

    switch (method) {
      case 'session/create':
        await this.createSession(ws, id);
        break;

      case 'tools/register':
        await this.registerTools(ws, id, params);
        break;

      case 'capabilities/grant':
        await this.grantCapabilities(ws, id, params);
        break;

      default:
        // Handle response to tool call
        if (message.result !== undefined || message.error !== undefined) {
          await this.routeToolResult(ws, message);
        }
    }
  }

  private async createSession(ws: WebSocket, requestId: string): Promise<void> {
    const sessionId = this.generateSessionId();
    const ttl = this.options.sessionTTL || 3600000;

    const session: Session = {
      id: sessionId,
      client: ws,
      agents: new Map(),
      tools: [],
      capabilities: new Set(),
      createdAt: Date.now(),
      expiresAt: Date.now() + ttl
    };

    this.sessions.set(sessionId, session);
    this.connectionToSession.set(ws, sessionId);

    this.send(ws, {
      jsonrpc: '2.0',
      id: requestId,
      result: { sessionId, expiresAt: session.expiresAt }
    });

    this.emit('session:created', { sessionId });
  }

  private async registerTools(
    ws: WebSocket,
    requestId: string,
    params: { tools: Tool[] }
  ): Promise<void> {
    const sessionId = this.connectionToSession.get(ws);
    const session = this.sessions.get(sessionId);

    if (!session) {
      throw new Error('No session');
    }

    session.tools = params.tools;

    // Notify agents of tool change
    for (const [agentId, agentWs] of session.agents) {
      this.send(agentWs, {
        jsonrpc: '2.0',
        method: 'notifications/tools/list_changed'
      });
    }

    this.send(ws, {
      jsonrpc: '2.0',
      id: requestId,
      result: { registered: params.tools.length }
    });
  }

  private handleAgentConnection(ws: WebSocket, req: any): void {
    // Extract session ID from URL query
    const url = new URL(req.url, `http://${req.headers.host}`);
    const sessionId = url.searchParams.get('session');

    if (!sessionId) {
      ws.close(4000, 'Session ID required');
      return;
    }

    const session = this.sessions.get(sessionId);
    if (!session) {
      ws.close(4001, 'Session not found');
      return;
    }

    const agentId = this.generateAgentId();
    session.agents.set(agentId, ws);
    this.connectionToSession.set(ws, sessionId);

    console.log(`Agent ${agentId} joined session ${sessionId}`);

    ws.on('message', async (data) => {
      try {
        const message = JSON.parse(data.toString());
        await this.handleAgentMessage(ws, agentId, message);
      } catch (error) {
        this.sendError(ws, null, error);
      }
    });

    ws.on('close', () => {
      session.agents.delete(agentId);
      this.connectionToSession.delete(ws);
      console.log(`Agent ${agentId} left session ${sessionId}`);
    });
  }

  private async handleAgentMessage(
    ws: WebSocket,
    agentId: string,
    message: any
  ): Promise<void> {
    const { method, params, id } = message;
    const sessionId = this.connectionToSession.get(ws);
    const session = this.sessions.get(sessionId);

    if (!session) {
      throw new Error('Session not found');
    }

    switch (method) {
      // MCP: Initialize
      case 'initialize':
        this.send(ws, {
          jsonrpc: '2.0',
          id,
          result: {
            protocolVersion: '2024-11-05',
            capabilities: { tools: { listChanged: true } },
            serverInfo: { name: 'btcp-server', version: '1.0.0' }
          }
        });
        break;

      // MCP: List tools
      case 'tools/list':
        const mcpTools = session.tools.map(t => ({
          name: t.name,
          description: t.description,
          inputSchema: t.inputSchema
        }));

        this.send(ws, {
          jsonrpc: '2.0',
          id,
          result: { tools: mcpTools }
        });
        break;

      // MCP: Call tool
      case 'tools/call':
        await this.routeToolCall(session, agentId, id, params);
        break;
    }
  }

  private async routeToolCall(
    session: Session,
    agentId: string,
    requestId: string,
    params: { name: string; arguments: any }
  ): Promise<void> {
    if (!session.client) {
      throw new Error('Client disconnected');
    }

    // Generate internal request ID for tracking
    const internalId = `${agentId}:${requestId}`;

    // Forward to client
    this.send(session.client, {
      jsonrpc: '2.0',
      id: internalId,
      method: 'tools/call',
      params: {
        name: params.name,
        arguments: params.arguments
      }
    });
  }

  private async routeToolResult(ws: WebSocket, message: any): Promise<void> {
    const sessionId = this.connectionToSession.get(ws);
    const session = this.sessions.get(sessionId);

    if (!session) return;

    // Parse internal ID to find agent
    const [agentId, originalId] = message.id.split(':');
    const agentWs = session.agents.get(agentId);

    if (!agentWs) return;

    // Translate to MCP format and send to agent
    if (message.error) {
      this.send(agentWs, {
        jsonrpc: '2.0',
        id: originalId,
        error: {
          code: -32603,
          message: message.error.message
        }
      });
    } else {
      this.send(agentWs, {
        jsonrpc: '2.0',
        id: originalId,
        result: {
          content: [{
            type: 'text',
            text: JSON.stringify(message.result)
          }]
        }
      });
    }
  }

  private handleClientDisconnect(ws: WebSocket): void {
    const sessionId = this.connectionToSession.get(ws);
    if (!sessionId) return;

    const session = this.sessions.get(sessionId);
    if (!session) return;

    // Notify agents
    for (const [agentId, agentWs] of session.agents) {
      agentWs.close(4002, 'Client disconnected');
    }

    // Cleanup
    this.sessions.delete(sessionId);
    this.connectionToSession.delete(ws);

    this.emit('session:terminated', { sessionId, reason: 'client_disconnect' });
  }

  private send(ws: WebSocket, message: any): void {
    ws.send(JSON.stringify(message));
  }

  private sendError(ws: WebSocket, id: string | null, error: Error): void {
    this.send(ws, {
      jsonrpc: '2.0',
      id,
      error: {
        code: -32603,
        message: error.message
      }
    });
  }

  private generateSessionId(): string {
    return crypto.randomUUID();
  }

  private generateAgentId(): string {
    return `agent-${crypto.randomUUID().slice(0, 8)}`;
  }
}
```

### Client SDK

```typescript
class BTCPClient {
  private ws: WebSocket;
  private sessionId: string | null = null;
  private pending: Map<string, { resolve: Function; reject: Function }>;
  private tools: Map<string, ToolImplementation>;

  constructor(private options: { serverUrl: string }) {
    this.pending = new Map();
    this.tools = new Map();
  }

  async connect(): Promise<{ sessionId: string }> {
    return new Promise((resolve, reject) => {
      this.ws = new WebSocket(this.options.serverUrl);

      this.ws.onopen = async () => {
        try {
          const result = await this.send('session/create', {});
          this.sessionId = result.sessionId;
          resolve({ sessionId: this.sessionId });
        } catch (error) {
          reject(error);
        }
      };

      this.ws.onmessage = (event) => {
        this.handleMessage(JSON.parse(event.data));
      };

      this.ws.onerror = (error) => {
        reject(error);
      };
    });
  }

  async registerManifest(manifest: Manifest, implementations: Record<string, Function>): Promise<void> {
    // Store implementations locally
    for (const tool of manifest.tools) {
      this.tools.set(tool.name, implementations[tool.name]);
    }

    // Register schemas with server
    await this.send('tools/register', { tools: manifest.tools });
  }

  private async handleMessage(message: any): Promise<void> {
    // Handle response to our request
    if (message.id && this.pending.has(message.id)) {
      const { resolve, reject } = this.pending.get(message.id);
      this.pending.delete(message.id);

      if (message.error) {
        reject(new Error(message.error.message));
      } else {
        resolve(message.result);
      }
      return;
    }

    // Handle tool call from agent (via server)
    if (message.method === 'tools/call') {
      await this.handleToolCall(message);
    }
  }

  private async handleToolCall(message: any): Promise<void> {
    const { name, arguments: args } = message.params;
    const implementation = this.tools.get(name);

    try {
      if (!implementation) {
        throw new Error(`Tool not found: ${name}`);
      }

      // Execute tool in sandbox (simplified here)
      const result = await implementation(args);

      // Send result back to server
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
          code: -32603,
          message: error.message
        }
      }));
    }
  }

  private send(method: string, params: any): Promise<any> {
    return new Promise((resolve, reject) => {
      const id = crypto.randomUUID();
      this.pending.set(id, { resolve, reject });

      this.ws.send(JSON.stringify({
        jsonrpc: '2.0',
        id,
        method,
        params
      }));
    });
  }
}
```

## Deployment

### Docker Deployment

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 8765 8766

CMD ["node", "server.js"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  btcp-server:
    build: .
    ports:
      - "8765:8765"  # Client connections
      - "8766:8766"  # Agent connections (MCP)
    environment:
      - BTCP_SESSION_TTL=3600
      - BTCP_LOG_LEVEL=info
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8765/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: btcp-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: btcp-server
  template:
    metadata:
      labels:
        app: btcp-server
    spec:
      containers:
      - name: btcp-server
        image: btcp/server:latest
        ports:
        - containerPort: 8765
          name: client
        - containerPort: 8766
          name: agent
        env:
        - name: BTCP_SESSION_STORE
          value: redis://redis:6379
---
apiVersion: v1
kind: Service
metadata:
  name: btcp-server
spec:
  selector:
    app: btcp-server
  ports:
  - name: client
    port: 8765
  - name: agent
    port: 8766
```

### Load Balancing

For WebSocket connections, use sticky sessions:

```nginx
upstream btcp_clients {
    ip_hash;
    server btcp-1:8765;
    server btcp-2:8765;
    server btcp-3:8765;
}

upstream btcp_agents {
    ip_hash;
    server btcp-1:8766;
    server btcp-2:8766;
    server btcp-3:8766;
}

server {
    listen 443 ssl;

    location /btcp/client {
        proxy_pass http://btcp_clients;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    location /btcp/agent {
        proxy_pass http://btcp_agents;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

## Monitoring

### Metrics

```typescript
class MetricsCollector {
  private metrics = {
    sessions: {
      active: 0,
      total: 0
    },
    connections: {
      clients: 0,
      agents: 0
    },
    messages: {
      received: 0,
      sent: 0,
      errors: 0
    },
    toolCalls: {
      total: 0,
      successful: 0,
      failed: 0,
      latency: [] as number[]
    }
  };

  recordToolCall(duration: number, success: boolean): void {
    this.metrics.toolCalls.total++;
    this.metrics.toolCalls.latency.push(duration);

    if (success) {
      this.metrics.toolCalls.successful++;
    } else {
      this.metrics.toolCalls.failed++;
    }
  }

  getMetrics(): object {
    return {
      ...this.metrics,
      toolCalls: {
        ...this.metrics.toolCalls,
        avgLatency: this.average(this.metrics.toolCalls.latency),
        p99Latency: this.percentile(this.metrics.toolCalls.latency, 99)
      }
    };
  }
}
```

### Health Endpoint

```typescript
server.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    uptime: process.uptime(),
    sessions: sessions.size,
    connections: {
      clients: clientConnections.size,
      agents: agentConnections.size
    }
  });
});
```

## Related

- [Server Overview](./index.md)
- [MCP Bridge](./mcp-bridge.md)
- [Session Management](./sessions.md)
