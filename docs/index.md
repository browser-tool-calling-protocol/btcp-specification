# Browser Tool Calling Protocol (BTCP)

BTCP is an open standard that enables AI agents to discover and invoke tools directly within browsers through client-defined interfaces.

## The Problem

Traditional AI tool integration requires server round-trips for every action. When an AI agent needs to interact with a web application, data must travel from the browser to a server, then to the AI, and back again. This creates:

- **Latency**: Every interaction requires multiple network hops
- **Context Loss**: Server-side tools lack access to browser state, DOM, and user session
- **Security Overhead**: Sensitive data must leave the browser to reach server-side tool handlers
- **Limited Capability**: Many browser-native operations simply cannot be performed server-side

## The Solution

BTCP takes a fundamentally different approach: **tools are defined and executed on the client side**. The browser itself becomes the tool execution environment, with AI agents calling tools that run locally within the user's browser session.

> If a user can interact with a web application, an AI agent should be able to interact with it too—with the same security and no server round-trips.

## Key Features

### Client-Side Execution
Tools run directly in the browser with native access to:
- DOM and page content
- Browser storage (localStorage, sessionStorage, IndexedDB)
- Web application state
- User session context

### Security First
- **Capability-based permissions**: Tools must declare required capabilities upfront
- **User consent**: Explicit approval required before tool access
- **Sandboxed execution**: Tools run in isolated environments
- **End-to-end encryption**: Sensitive data protected in transit

### Multi-Step Workflows
Execute complex workflows in a single request instead of sequential server round-trips. The agent can perform multiple browser operations atomically.

### Protocol Flexibility
BTCP supports multiple execution environments:
- **Web Workers**: Isolated thread-based execution
- **iframes**: DOM-isolated sandboxes
- **SES Compartments**: Secure ECMAScript compartments
- **WebAssembly**: Hardware-isolated execution

## How It Works

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                Browser                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐                 │
│  │  Web App    │    │ BTCP Client │    │  Tool Sandbox   │                 │
│  │             │◄──►│             │◄──►│                 │                 │
│  │  (DOM,      │    │  (Schema    │    │  (Isolated      │                 │
│  │   State)    │    │   Registry) │    │   Execution)    │                 │
│  └─────────────┘    └──────┬──────┘    └─────────────────┘                 │
│                            │                                                │
└────────────────────────────┼────────────────────────────────────────────────┘
                             │ BTCP
                             │ (WebSocket)
                    ┌────────▼────────┐
                    │   BTCP Server   │
                    │                 │
                    │  (Transport     │
                    │   Relay Only)   │
                    └────────┬────────┘
                             │ MCP
                             │ (Stdio/WebSocket/SSE)
                    ┌────────▼────────┐
                    │    AI Agent     │
                    │                 │
                    │ (Claude, GPT,   │
                    │  Custom Agent)  │
                    └─────────────────┘
```

1. **Browser Client** registers tools and executes them in sandboxed environments
2. **BTCP Server** acts as a transport relay between browser and agents (MCP-compatible)
3. **AI Agent** connects via standard MCP protocol to discover and call tools
4. **Tool execution** happens entirely in the browser—server only routes messages
5. **Results** flow back through the server to the requesting agent

### Architecture Highlights

- **Server is a pure relay**: No tool execution, no data processing—just message routing
- **MCP-compatible**: AI agents connect using standard MCP protocol
- **Session-based**: Browser creates session, agents join with session ID
- **Tools execute client-side**: All tool logic runs in browser sandboxes

## Complementary to MCP

BTCP is designed to work alongside the Model Context Protocol (MCP):

| Aspect | MCP | BTCP |
|--------|-----|------|
| Execution | Server-side | Client-side (browser) |
| Context | Server resources | Browser state, DOM |
| Use Case | Backend operations | Frontend interactions |
| Latency | Network round-trips | Local execution |

Use MCP for server operations (databases, file systems, external APIs) and BTCP for browser-side tool execution—in the same agent.

## Getting Started

- **[Introduction](./introduction.md)**: Deep dive into BTCP concepts
- **[For Tool Providers](./for-tool-providers.md)**: Create BTCP-compliant tools
- **[For Tool Callers](./for-tool-callers.md)**: Integrate BTCP into your AI agent
- **[BTCP Server](./server/index.md)**: Set up the MCP-compatible relay server
- **[Security Model](./security.md)**: Understand BTCP's security architecture
- **[API Reference](./api/core/manifest.md)**: Complete schema specifications

## License

BTCP is released under the [Mozilla Public License 2.0 (MPL-2.0)](https://mozilla.org/MPL/2.0/).
