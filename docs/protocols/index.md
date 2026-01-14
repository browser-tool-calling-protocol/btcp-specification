# Protocols Overview

BTCP supports multiple execution environments (sandboxes) for running tools securely within the browser. This document provides an overview of available protocols.

## Execution Environments

BTCP tools run within sandboxed environments that enforce capability restrictions. Each sandbox type offers different trade-offs between isolation, performance, and compatibility.

### Available Sandboxes

| Sandbox | Isolation Level | Performance | Browser Support | Best For |
|---------|-----------------|-------------|-----------------|----------|
| [Web Worker](./worker.md) | Medium | Good | Excellent | General purpose |
| [iframe](./iframe.md) | Medium | Good | Excellent | DOM-heavy tools |
| [SES](./ses.md) | High | Excellent | Good | Security-critical |
| [WebAssembly](./wasm.md) | Very High | Excellent | Excellent | Compute-intensive |

## Transport Protocols

BTCP supports multiple transport mechanisms for communication between AI agents and the browser.

### WebSocket

Direct WebSocket connection for real-time bidirectional communication.

```
Agent ◄──── WSS ────► BTCP Client (Browser)
```

**Use Cases:**
- Local AI agents
- Desktop applications
- Development environments

[WebSocket Protocol Details](./websocket.md)

### Extension Bridge

Communication via browser extension messaging.

```
Agent ◄──── Extension API ────► BTCP Client (Browser)
```

**Use Cases:**
- Browser extension-based agents
- Cross-origin communication
- Enhanced security context

[Extension Bridge Details](./extension.md)

### Cloud Relay

Communication through a cloud relay server.

```
Agent ◄──── HTTPS ────► Cloud Relay ◄──── WSS ────► Browser
```

**Use Cases:**
- Remote AI agents
- Cloud-hosted agents
- Multi-device scenarios

[Cloud Relay Details](./cloud.md)

## Message Protocol

All transports use JSON-RPC 2.0 for message formatting.

### Request Format

```json
{
  "jsonrpc": "2.0",
  "id": "unique-request-id",
  "method": "method/name",
  "params": {
    "key": "value"
  }
}
```

### Response Format

**Success:**
```json
{
  "jsonrpc": "2.0",
  "id": "unique-request-id",
  "result": {
    "data": "value"
  }
}
```

**Error:**
```json
{
  "jsonrpc": "2.0",
  "id": "unique-request-id",
  "error": {
    "code": -32600,
    "message": "Invalid Request",
    "data": {
      "details": "Additional error information"
    }
  }
}
```

## Standard Methods

### Discovery Methods

| Method | Description |
|--------|-------------|
| `manifests/list` | List all registered manifests |
| `tools/list` | List all available tools |
| `tools/get` | Get details of a specific tool |
| `capabilities/list` | List granted capabilities |

### Execution Methods

| Method | Description |
|--------|-------------|
| `tools/call` | Execute a tool |
| `tools/callBatch` | Execute multiple tools |
| `capabilities/request` | Request capability grants |

### Session Methods

| Method | Description |
|--------|-------------|
| `session/info` | Get session information |
| `session/ping` | Keepalive ping |

## Error Codes

Standard JSON-RPC error codes:

| Code | Message | Description |
|------|---------|-------------|
| -32700 | Parse error | Invalid JSON |
| -32600 | Invalid Request | Request structure invalid |
| -32601 | Method not found | Unknown method |
| -32602 | Invalid params | Parameter validation failed |
| -32603 | Internal error | Server error |

BTCP-specific error codes:

| Code | Message | Description |
|------|---------|-------------|
| -32000 | Tool not found | Requested tool doesn't exist |
| -32001 | Capability denied | Required capability not granted |
| -32002 | Execution timeout | Tool exceeded timeout |
| -32003 | Sandbox error | Sandbox failed to execute |
| -32004 | Validation error | Input validation failed |

## Selecting a Protocol

### Decision Matrix

| Requirement | Recommended |
|-------------|-------------|
| Lowest latency | WebSocket |
| Maximum security | SES + Extension Bridge |
| Remote access | Cloud Relay |
| Browser compatibility | Web Worker + WebSocket |
| Heavy computation | WebAssembly |

### Configuration Example

```json
{
  "btcp": "1.0",
  "name": "my-tools",
  "config": {
    "sandbox": "worker",
    "transport": "websocket",
    "timeout": 30000
  }
}
```

## Related

- [Web Worker Protocol](./worker.md)
- [iframe Protocol](./iframe.md)
- [SES Protocol](./ses.md)
- [WebAssembly Protocol](./wasm.md)
- [WebSocket Transport](./websocket.md)
- [Security Model](../security.md)
