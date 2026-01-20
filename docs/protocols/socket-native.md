# Socket Native Transport

Socket Native provides the lowest-latency transport option for BTCP, using Unix domain sockets (on Unix-like systems) or named pipes (on Windows) for same-machine inter-process communication.

## Overview

Socket Native bypasses the TCP/IP network stack entirely, making it ideal for scenarios where the AI agent and browser client run on the same machine.

```
┌──────────────────┐                      ┌──────────────────┐
│                  │                      │                  │
│    AI Agent      │◄──── Unix Socket ───►│   BTCP Server    │
│                  │      /Named Pipe     │                  │
│                  │                      │                  │
└──────────────────┘                      └──────────────────┘
         │                                         │
         │              Same Machine               │
         └─────────────────────────────────────────┘
```

## Advantages

| Feature | Socket Native | WebSocket (TCP) |
|---------|---------------|-----------------|
| Latency | ~10μs | ~100μs |
| Throughput | Higher | Lower |
| Port Management | Not required | Required |
| Firewall Issues | None | Possible |
| Network Stack | Bypassed | Full TCP/IP |
| Security | File permissions | Network ACLs |

### Key Benefits

1. **Ultra-low latency** - No TCP handshake or network stack overhead
2. **No port conflicts** - Uses file paths instead of port numbers
3. **Built-in security** - Leverages file system permissions
4. **Better performance** - Higher throughput for large payloads
5. **Simpler networking** - No firewall or NAT traversal concerns

## When to Use

Socket Native is recommended for:

- Local development environments
- Desktop AI agent applications
- Same-machine MCP server deployments
- Performance-critical tool execution
- Scenarios requiring minimal latency

## Configuration

### Server Configuration

```yaml
# btcp-server.yaml
server:
  transport: socket-native
  socket:
    # Unix domain socket path (Linux/macOS)
    path: /tmp/btcp.sock
    # Or for Windows named pipes
    # path: \\.\pipe\btcp

    # Socket file permissions (Unix only)
    mode: 0660

    # Optional: cleanup stale socket on start
    cleanup: true

sessions:
  ttl: 3600
```

### Environment Variables

```bash
# Unix domain socket
BTCP_TRANSPORT=socket-native
BTCP_SOCKET_PATH=/tmp/btcp.sock
BTCP_SOCKET_MODE=0660

# Windows named pipe
BTCP_TRANSPORT=socket-native
BTCP_SOCKET_PATH=\\.\pipe\btcp
```

### Programmatic Configuration

```typescript
import { BTCPServer } from '@btcp/server';

const server = new BTCPServer({
  transport: 'socket-native',
  socket: {
    path: '/tmp/btcp.sock',
    mode: 0o660,
    cleanup: true
  }
});

await server.start();
```

## Implementation

### Server Implementation (Node.js)

```typescript
import { createServer, Socket } from 'net';
import { unlinkSync, existsSync, chmodSync } from 'fs';
import { EventEmitter } from 'events';

interface SocketServerOptions {
  path: string;
  mode?: number;
  cleanup?: boolean;
}

class SocketNativeServer extends EventEmitter {
  private server: ReturnType<typeof createServer>;
  private connections: Map<string, Socket> = new Map();

  constructor(private options: SocketServerOptions) {
    super();
  }

  async start(): Promise<void> {
    // Cleanup stale socket file
    if (this.options.cleanup && existsSync(this.options.path)) {
      unlinkSync(this.options.path);
    }

    return new Promise((resolve, reject) => {
      this.server = createServer((socket) => {
        this.handleConnection(socket);
      });

      this.server.on('error', (err) => {
        reject(err);
      });

      this.server.listen(this.options.path, () => {
        // Set socket file permissions
        if (this.options.mode) {
          chmodSync(this.options.path, this.options.mode);
        }

        console.log(`Socket Native server listening on ${this.options.path}`);
        resolve();
      });
    });
  }

  private handleConnection(socket: Socket): void {
    const connectionId = this.generateId();
    this.connections.set(connectionId, socket);

    let buffer = '';

    socket.on('data', (data) => {
      buffer += data.toString();

      // Handle newline-delimited JSON messages
      const lines = buffer.split('\n');
      buffer = lines.pop() || '';

      for (const line of lines) {
        if (line.trim()) {
          try {
            const message = JSON.parse(line);
            this.emit('message', connectionId, message);
          } catch (err) {
            this.sendError(connectionId, null, 'Parse error');
          }
        }
      }
    });

    socket.on('close', () => {
      this.connections.delete(connectionId);
      this.emit('disconnect', connectionId);
    });

    socket.on('error', (err) => {
      console.error(`Socket error [${connectionId}]:`, err.message);
      this.connections.delete(connectionId);
    });

    this.emit('connect', connectionId);
  }

  send(connectionId: string, message: object): void {
    const socket = this.connections.get(connectionId);
    if (socket && !socket.destroyed) {
      socket.write(JSON.stringify(message) + '\n');
    }
  }

  private sendError(connectionId: string, id: string | null, message: string): void {
    this.send(connectionId, {
      jsonrpc: '2.0',
      id,
      error: { code: -32700, message }
    });
  }

  private generateId(): string {
    return `conn-${Date.now()}-${Math.random().toString(36).slice(2)}`;
  }

  async stop(): Promise<void> {
    for (const socket of this.connections.values()) {
      socket.destroy();
    }
    this.connections.clear();

    return new Promise((resolve) => {
      this.server.close(() => {
        // Cleanup socket file
        if (existsSync(this.options.path)) {
          unlinkSync(this.options.path);
        }
        resolve();
      });
    });
  }
}
```

### Client Implementation (Node.js)

```typescript
import { connect, Socket } from 'net';
import { EventEmitter } from 'events';

interface SocketClientOptions {
  path: string;
  reconnect?: boolean;
  reconnectInterval?: number;
}

class SocketNativeClient extends EventEmitter {
  private socket: Socket | null = null;
  private pending: Map<string, { resolve: Function; reject: Function }> = new Map();
  private reconnecting = false;

  constructor(private options: SocketClientOptions) {
    super();
  }

  async connect(): Promise<void> {
    return new Promise((resolve, reject) => {
      this.socket = connect(this.options.path);

      let buffer = '';

      this.socket.on('connect', () => {
        this.reconnecting = false;
        this.emit('connect');
        resolve();
      });

      this.socket.on('data', (data) => {
        buffer += data.toString();

        const lines = buffer.split('\n');
        buffer = lines.pop() || '';

        for (const line of lines) {
          if (line.trim()) {
            try {
              const message = JSON.parse(line);
              this.handleMessage(message);
            } catch (err) {
              console.error('Failed to parse message:', err);
            }
          }
        }
      });

      this.socket.on('close', () => {
        this.emit('disconnect');
        if (this.options.reconnect && !this.reconnecting) {
          this.scheduleReconnect();
        }
      });

      this.socket.on('error', (err) => {
        if (!this.socket?.connecting) {
          this.emit('error', err);
        }
        reject(err);
      });
    });
  }

  private handleMessage(message: any): void {
    // Handle response
    if (message.id && this.pending.has(message.id)) {
      const { resolve, reject } = this.pending.get(message.id)!;
      this.pending.delete(message.id);

      if (message.error) {
        reject(new Error(message.error.message));
      } else {
        resolve(message.result);
      }
      return;
    }

    // Handle notification/request from server
    this.emit('message', message);
  }

  async request(method: string, params?: object): Promise<any> {
    return new Promise((resolve, reject) => {
      const id = `req-${Date.now()}-${Math.random().toString(36).slice(2)}`;
      this.pending.set(id, { resolve, reject });

      const message = {
        jsonrpc: '2.0',
        id,
        method,
        params
      };

      this.socket?.write(JSON.stringify(message) + '\n');

      // Timeout
      setTimeout(() => {
        if (this.pending.has(id)) {
          this.pending.delete(id);
          reject(new Error('Request timeout'));
        }
      }, 30000);
    });
  }

  send(message: object): void {
    this.socket?.write(JSON.stringify(message) + '\n');
  }

  private scheduleReconnect(): void {
    this.reconnecting = true;
    const interval = this.options.reconnectInterval || 1000;

    setTimeout(async () => {
      try {
        await this.connect();
      } catch (err) {
        this.scheduleReconnect();
      }
    }, interval);
  }

  disconnect(): void {
    this.options.reconnect = false;
    this.socket?.destroy();
    this.socket = null;
  }
}
```

## Usage Examples

### Basic Server Setup

```typescript
import { BTCPServer } from '@btcp/server';

const server = new BTCPServer({
  transport: 'socket-native',
  socket: {
    path: '/tmp/btcp.sock'
  }
});

server.on('session:created', ({ sessionId }) => {
  console.log(`Session created: ${sessionId}`);
});

await server.start();
```

### AI Agent Connection

```typescript
import { Client } from '@modelcontextprotocol/sdk/client';
import { SocketNativeTransport } from '@btcp/transport-socket';

const client = new Client({
  name: 'my-agent',
  version: '1.0.0'
});

// Connect via Unix socket
const transport = new SocketNativeTransport({
  path: '/tmp/btcp.sock',
  sessionId: 'session-123'
});

await client.connect(transport);

// List and call tools
const tools = await client.listTools();
const result = await client.callTool('getPageContent', { selector: 'main' });
```

### Combined WebSocket + Socket Native

For maximum flexibility, run both transports simultaneously:

```typescript
import { BTCPServer } from '@btcp/server';

const server = new BTCPServer({
  // WebSocket for remote/browser clients
  clientPort: 8765,

  // Socket Native for local AI agents
  transport: 'socket-native',
  socket: {
    path: '/tmp/btcp.sock'
  }
});

await server.start();
// Browser clients connect via ws://localhost:8765
// Local agents connect via /tmp/btcp.sock
```

## Platform-Specific Notes

### Linux/macOS

Unix domain sockets are created as special files in the filesystem:

```bash
# Default locations
/tmp/btcp.sock           # Temporary (cleared on reboot)
/run/btcp/btcp.sock      # Runtime directory
~/.btcp/btcp.sock        # User-specific

# Check socket status
ls -la /tmp/btcp.sock
# srw-rw---- 1 user group 0 Jan 20 10:00 /tmp/btcp.sock

# Test connection with netcat
nc -U /tmp/btcp.sock
```

**File Permissions:**

```typescript
// Restrict to owner only
socket: { path: '/tmp/btcp.sock', mode: 0o600 }

// Owner and group
socket: { path: '/tmp/btcp.sock', mode: 0o660 }

// World readable (not recommended)
socket: { path: '/tmp/btcp.sock', mode: 0o666 }
```

### Windows

Windows uses named pipes instead of Unix sockets:

```typescript
const server = new BTCPServer({
  transport: 'socket-native',
  socket: {
    path: '\\\\.\\pipe\\btcp'
  }
});
```

Named pipe paths follow the format: `\\.\pipe\<name>`

```powershell
# List named pipes
Get-ChildItem -Path '\\.\pipe\' | Where-Object { $_.Name -like '*btcp*' }
```

### Abstract Sockets (Linux)

Linux supports abstract sockets that don't create filesystem entries:

```typescript
const server = new BTCPServer({
  transport: 'socket-native',
  socket: {
    // Prefix with null byte for abstract socket
    path: '\0btcp-server',
    abstract: true
  }
});
```

Benefits:
- Automatically cleaned up when process exits
- No filesystem permission issues
- Can't be accidentally deleted

## Security Considerations

### File Permissions

Unix socket security relies on filesystem permissions:

```typescript
// Recommended: restrict to current user
socket: {
  path: '/tmp/btcp.sock',
  mode: 0o600  // Owner read/write only
}

// For shared access with specific group
socket: {
  path: '/run/btcp/btcp.sock',
  mode: 0o660,  // Owner and group read/write
  group: 'btcp-users'
}
```

### Socket Location

Choose socket paths carefully:

| Location | Security | Notes |
|----------|----------|-------|
| `/tmp` | Low | Shared, tmpwatch may delete |
| `/run/user/$UID` | High | User-specific runtime dir |
| `~/.btcp/` | Medium | User home directory |
| `/run/btcp/` | Medium | System service directory |

### Process Isolation

For additional security, combine with process isolation:

```typescript
// Server validates connecting process
server.on('connect', async (connectionId, socket) => {
  const credentials = socket.getPeerCredentials();

  // Verify connecting process UID
  if (credentials.uid !== process.getuid()) {
    socket.destroy();
    return;
  }

  // Optional: verify process name via /proc
  const processName = await getProcessName(credentials.pid);
  if (!allowedProcesses.includes(processName)) {
    socket.destroy();
    return;
  }
});
```

## Troubleshooting

### Common Issues

**Socket file already exists:**
```
Error: EADDRINUSE: address already in use /tmp/btcp.sock
```

Solution: Enable cleanup or manually remove:
```typescript
socket: { path: '/tmp/btcp.sock', cleanup: true }
```

**Permission denied:**
```
Error: EACCES: permission denied /tmp/btcp.sock
```

Solution: Check file permissions and ownership:
```bash
ls -la /tmp/btcp.sock
chmod 660 /tmp/btcp.sock
```

**Connection refused:**
```
Error: ECONNREFUSED: connection refused /tmp/btcp.sock
```

Solution: Verify server is running:
```bash
# Check if socket exists and server is listening
ss -x | grep btcp
```

### Debugging

Enable verbose logging:

```typescript
const server = new BTCPServer({
  transport: 'socket-native',
  socket: { path: '/tmp/btcp.sock' },
  logging: {
    level: 'debug',
    transport: true  // Log all transport messages
  }
});
```

## Performance Tuning

### Buffer Sizes

```typescript
const server = new BTCPServer({
  transport: 'socket-native',
  socket: {
    path: '/tmp/btcp.sock',
    // Increase for large payloads
    highWaterMark: 64 * 1024,  // 64KB
    // Disable Nagle's algorithm (always recommended)
    noDelay: true
  }
});
```

### Connection Pooling

For high-throughput scenarios:

```typescript
class ConnectionPool {
  private connections: SocketNativeClient[] = [];
  private roundRobin = 0;

  constructor(private path: string, private size: number) {}

  async initialize(): Promise<void> {
    for (let i = 0; i < this.size; i++) {
      const client = new SocketNativeClient({ path: this.path });
      await client.connect();
      this.connections.push(client);
    }
  }

  getConnection(): SocketNativeClient {
    const conn = this.connections[this.roundRobin];
    this.roundRobin = (this.roundRobin + 1) % this.size;
    return conn;
  }
}
```

## Related

- [Protocols Overview](./index.md)
- [WebSocket Transport](./websocket.md)
- [Server Implementation](../server/implementation.md)
- [Security Model](../security.md)
