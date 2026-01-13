# Browser Tool Calling Protocol (BTCP) Specification

[![License: MPL 2.0](https://img.shields.io/badge/License-MPL%202.0-brightgreen.svg)](https://opensource.org/licenses/MPL-2.0)

The official specification for the Browser Tool Calling Protocol (BTCP) — an open standard enabling AI agents to discover and invoke tools directly within browsers through client-defined interfaces.

## What is BTCP?

BTCP is a protocol that lets AI agents call browser tools securely. Unlike server-side tool protocols, BTCP tools are **defined and executed on the client side**, providing:

- **Direct browser access**: Native access to DOM, storage, and web application state
- **No server round-trips**: Tools execute locally in the browser
- **Capability-based security**: Explicit permissions with user consent
- **Sandboxed execution**: Tools run in isolated environments

## Quick Links

| Resource | Description |
|----------|-------------|
| [Overview](./docs/index.md) | Introduction to BTCP |
| [For Tool Providers](./docs/for-tool-providers.md) | Create BTCP-compliant tools |
| [For Tool Callers](./docs/for-tool-callers.md) | Integrate BTCP into AI agents |
| [Security Model](./docs/security.md) | Capability system and sandboxing |
| [API Reference](./docs/api/core/manifest.md) | Schema specifications |

## How It Works

```
┌─────────────────────────────────────────────────────────────┐
│                         Browser                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐  │
│  │  Web App    │    │ BTCP Client │    │  Tool Sandbox   │  │
│  │             │◄──►│             │◄──►│                 │  │
│  │  (DOM,      │    │  (Schema    │    │  (Isolated      │  │
│  │   State)    │    │   Registry) │    │   Execution)    │  │
│  └─────────────┘    └──────┬──────┘    └─────────────────┘  │
└────────────────────────────┼─────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │    AI Agent     │
                    └─────────────────┘
```

1. **Tool Providers** expose tools via BTCP manifests
2. **BTCP Client** manages tool registry and permissions
3. **AI Agent** discovers and calls tools via JSON-RPC
4. **Sandbox** executes tools with capability restrictions

## Example

### Tool Manifest

```json
{
  "btcp": "1.0",
  "name": "page-tools",
  "version": "1.0.0",
  "tools": [
    {
      "name": "getPageContent",
      "description": "Extracts text content from the current page",
      "inputSchema": {
        "type": "object",
        "properties": {
          "selector": { "type": "string", "default": "main" }
        }
      },
      "capabilities": ["dom:read"]
    }
  ],
  "capabilities": ["dom:read"]
}
```

### Calling from an AI Agent

```javascript
import { BTCPAgent } from '@btcp/agent-sdk';

const agent = new BTCPAgent({ transport: 'websocket' });
await agent.connect();

// Request capability
await agent.requestCapabilities(['dom:read']);

// Call tool
const result = await agent.callTool('getPageContent', {
  selector: 'article'
});

console.log(result.content);
```

## Documentation Structure

```
docs/
├── index.md                    # Overview
├── introduction.md             # Detailed introduction
├── for-tool-providers.md       # Provider guide
├── for-tool-callers.md         # Caller guide
├── security.md                 # Security model
├── implementation.md           # Best practices
├── api/
│   └── core/
│       ├── manifest.md         # Manifest schema
│       ├── tool.md             # Tool schema
│       └── capabilities.md     # Capability reference
└── protocols/
    ├── index.md                # Protocols overview
    ├── worker.md               # Web Worker sandbox
    ├── iframe.md               # iframe sandbox
    ├── ses.md                  # SES sandbox
    └── wasm.md                 # WebAssembly sandbox
```

## Key Features

### Security
- **Capability-based permissions**: Tools declare required capabilities
- **User consent**: Explicit approval for sensitive operations
- **Sandboxed execution**: Multiple isolation options (Worker, iframe, SES, Wasm)
- **Input validation**: JSON Schema enforcement

### Flexibility
- **Multiple transports**: WebSocket, browser extension, cloud relay
- **Multiple sandboxes**: Choose isolation level per use case
- **Context-aware**: Dynamic tool registration based on page state

### Interoperability
- **Complements MCP**: Use alongside Model Context Protocol
- **LLM integration**: Convert to OpenAI/Anthropic tool formats
- **Standard protocols**: JSON-RPC 2.0, JSON Schema

## BTCP vs MCP

| Aspect | MCP | BTCP |
|--------|-----|------|
| Execution | Server-side | Client-side (browser) |
| Context | Server resources | Browser state, DOM |
| Use Case | Backend operations | Frontend interactions |
| Latency | Network round-trips | Local execution |

BTCP is designed to complement MCP — use both together for full-stack AI tool integration.

## Contributing

We welcome contributions! Please see our contributing guidelines:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

### Areas for Contribution

- Specification improvements
- Documentation enhancements
- Reference implementations
- Security reviews
- Test cases

## Related Projects

- [BTCP Client SDK](https://github.com/browser-tool-calling-protocol/btcp-client) - Browser client implementation
- [BTCP Agent SDK](https://github.com/browser-tool-calling-protocol/btcp-agent) - AI agent integration library
- [BTCP Chrome Extension](https://github.com/browser-tool-calling-protocol/btcp-extension) - Browser extension

## License

This specification is released under the [Mozilla Public License 2.0 (MPL-2.0)](LICENSE).

## Community

- [GitHub Discussions](https://github.com/browser-tool-calling-protocol/btcp-specification/discussions)
- [Discord](https://discord.gg/btcp)
- [Twitter](https://twitter.com/btcp_protocol)

---

**Browser Tool Calling Protocol** — Enabling AI agents to interact with the web, securely.
