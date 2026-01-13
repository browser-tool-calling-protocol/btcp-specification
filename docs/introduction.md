# Introduction to BTCP

The Browser Tool Calling Protocol (BTCP) is a specification for enabling AI agents to interact with web applications through client-side tool execution. This document provides a comprehensive introduction to BTCP's architecture, concepts, and design philosophy.

## Background

### The Evolution of AI Tool Calling

AI agents have become increasingly capable of using tools to accomplish tasks. The Model Context Protocol (MCP) established a standard for server-side tool integration, enabling agents to interact with databases, file systems, and external APIs.

However, a significant gap remained: **browser-based interactions**. Web applications contain rich state, complex UIs, and user-specific context that server-side tools cannot access. BTCP fills this gap by bringing tool execution into the browser itself.

### Why Client-Side?

Consider an AI assistant helping a user with a web-based spreadsheet:

**Server-side approach (traditional):**
1. User asks to "format column A as currency"
2. Request goes to AI server
3. AI server calls formatting API
4. API must authenticate, find the document, make changes
5. Changes sync back to browser
6. User sees result after multiple round-trips

**Client-side approach (BTCP):**
1. User asks to "format column A as currency"
2. Request goes to AI (local or cloud)
3. AI calls BTCP tool in browser
4. Tool directly manipulates DOM/application state
5. User sees result immediately

The client-side approach is faster, more contextual, and more secure—sensitive document data never leaves the browser.

## Core Concepts

### Manifests

A **Manifest** is a BTCP document that describes a collection of tools. It includes:

- **Metadata**: Name, version, description, and provider information
- **Tools**: List of available tools with their schemas
- **Capabilities**: Required permissions for the tool collection
- **Configuration**: Runtime settings and options

```json
{
  "btcp": "1.0",
  "name": "spreadsheet-tools",
  "version": "1.0.0",
  "description": "Tools for interacting with spreadsheet applications",
  "tools": [...],
  "capabilities": ["dom:read", "dom:write", "storage:read"]
}
```

### Tools

A **Tool** is a callable function exposed to AI agents. Each tool has:

- **Name**: Unique identifier within the manifest
- **Description**: Human and AI-readable explanation
- **Input Schema**: JSON Schema defining expected parameters
- **Output Schema**: JSON Schema defining return value structure
- **Capabilities**: Specific permissions required by this tool

```json
{
  "name": "formatCells",
  "description": "Format selected cells with the specified style",
  "inputSchema": {
    "type": "object",
    "properties": {
      "range": { "type": "string", "description": "Cell range (e.g., 'A1:B10')" },
      "format": { "type": "string", "enum": ["currency", "percentage", "date"] }
    },
    "required": ["range", "format"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "success": { "type": "boolean" },
      "cellsAffected": { "type": "integer" }
    }
  },
  "capabilities": ["dom:write"]
}
```

### Capabilities

**Capabilities** are permissions that tools must declare and users must grant. BTCP uses a capability-based security model where:

1. Tools declare what they need upfront
2. Users see exactly what permissions are requested
3. Execution is blocked if capabilities aren't granted

Standard capability categories:

| Category | Examples | Description |
|----------|----------|-------------|
| `dom` | `dom:read`, `dom:write` | Access to page content |
| `storage` | `storage:read`, `storage:write` | Browser storage APIs |
| `network` | `network:fetch`, `network:websocket` | Network requests |
| `clipboard` | `clipboard:read`, `clipboard:write` | Clipboard access |
| `media` | `media:camera`, `media:microphone` | Media devices |

### Sandboxes

Tools execute within **Sandboxes**—isolated environments that enforce capability restrictions. BTCP supports multiple sandbox implementations:

- **Web Workers**: Separate thread with message-passing
- **iframes**: DOM-isolated browsing context
- **SES Compartments**: Secure ECMAScript with frozen intrinsics
- **WebAssembly Isolates**: Hardware-level memory isolation

## Architecture

### Component Overview

```
┌────────────────────────────────────────────────────────────────┐
│                           Browser                               │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │ Web          │  │ BTCP         │  │ Sandbox Runtime       │  │
│  │ Application  │  │ Client       │  │                       │  │
│  │              │  │              │  │  ┌─────────────────┐  │  │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │  │ Tool Instance   │  │  │
│  │ │  State   │◄┼──┼─┤ Registry │ │  │  │                 │  │  │
│  │ └──────────┘ │  │ └──────────┘ │  │  │ Capabilities:   │  │  │
│  │              │  │              │  │  │ [dom:read]      │  │  │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │  └─────────────────┘  │  │
│  │ │   DOM    │◄┼──┼─┤ Executor │◄┼──┼──────────────────────►│  │
│  │ └──────────┘ │  │ └──────────┘ │  │                       │  │
│  │              │  │              │  │  ┌─────────────────┐  │  │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │  │ Tool Instance   │  │  │
│  │ │ Storage  │◄┼──┼─┤ Security │ │  │  │                 │  │  │
│  │ └──────────┘ │  │ └──────────┘ │  │  │ Capabilities:   │  │  │
│  └──────────────┘  └──────┬───────┘  │  │ [storage:write] │  │  │
│                           │          │  └─────────────────┘  │  │
│                           │          └───────────────────────┘  │
└───────────────────────────┼──────────────────────────────────────┘
                            │
                   ┌────────▼────────┐
                   │                 │
                   │    AI Agent     │
                   │                 │
                   │  ┌───────────┐  │
                   │  │ BTCP SDK  │  │
                   │  └───────────┘  │
                   │                 │
                   └─────────────────┘
```

### Request Flow

1. **Discovery**: Agent queries the BTCP Client for available tools
2. **Selection**: Agent chooses tools based on manifest schemas
3. **Request**: Agent sends tool call with parameters
4. **Validation**: BTCP Client validates parameters against schema
5. **Permission Check**: Security layer verifies capabilities are granted
6. **Sandbox Creation**: Executor creates isolated environment
7. **Execution**: Tool runs with access to permitted resources
8. **Response**: Results returned to agent

### Message Format

BTCP uses JSON-RPC 2.0 for communication:

**Request:**
```json
{
  "jsonrpc": "2.0",
  "id": "req-001",
  "method": "tools/call",
  "params": {
    "name": "formatCells",
    "arguments": {
      "range": "A1:A10",
      "format": "currency"
    }
  }
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "id": "req-001",
  "result": {
    "success": true,
    "cellsAffected": 10
  }
}
```

## Design Principles

### 1. Security by Default

Every aspect of BTCP is designed with security in mind:
- No implicit permissions
- Explicit capability declarations
- Sandboxed execution
- User consent requirements

### 2. Progressive Enhancement

BTCP works with varying levels of browser support:
- Core functionality in any modern browser
- Enhanced isolation with Web Workers
- Maximum security with SES or WebAssembly

### 3. Interoperability

BTCP is designed to complement existing protocols:
- Works alongside MCP for full-stack AI integration
- Compatible with standard web APIs
- Extensible for future protocols

### 4. Developer Experience

Creating and consuming BTCP tools should be straightforward:
- JSON Schema for tool definitions
- TypeScript support out of the box
- Clear error messages
- Comprehensive documentation

## Use Cases

### Web Application Automation
AI agents can automate complex workflows within web applications—filling forms, navigating interfaces, and extracting data.

### Accessibility Enhancement
BTCP tools can provide alternative interaction methods for users with disabilities, with AI agents mediating complex UI patterns.

### Testing and QA
Automated testing frameworks can use BTCP to create AI-driven test scenarios that adapt to UI changes.

### Browser Extension Integration
Extensions can expose functionality via BTCP, allowing AI agents to leverage installed browser capabilities.

## Next Steps

- **[For Tool Providers](./for-tool-providers.md)**: Learn how to create BTCP tools
- **[For Tool Callers](./for-tool-callers.md)**: Integrate BTCP into your AI agent
- **[Security Model](./security.md)**: Deep dive into BTCP security
- **[API Reference](./api/core/manifest.md)**: Complete schema documentation
