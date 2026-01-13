# BTCP for Tool Providers

This guide explains how to create BTCP-compliant tools that AI agents can discover and invoke within browsers.

## Overview

As a tool provider, you create a **BTCP Manifest**—a standardized JSON document that describes your tools, their inputs/outputs, and required capabilities. This enables AI agents to understand and use your tools without custom integration.

## Quick Start

### 1. Define Your Manifest

Create a manifest file describing your tools:

```json
{
  "btcp": "1.0",
  "name": "my-web-tools",
  "version": "1.0.0",
  "description": "Tools for interacting with my web application",
  "provider": {
    "name": "My Company",
    "url": "https://example.com"
  },
  "tools": [
    {
      "name": "getPageContent",
      "description": "Extracts the main content from the current page",
      "inputSchema": {
        "type": "object",
        "properties": {
          "selector": {
            "type": "string",
            "description": "CSS selector for content area (default: 'main')"
          },
          "format": {
            "type": "string",
            "enum": ["text", "html", "markdown"],
            "default": "text"
          }
        }
      },
      "outputSchema": {
        "type": "object",
        "properties": {
          "content": { "type": "string" },
          "wordCount": { "type": "integer" }
        },
        "required": ["content"]
      },
      "capabilities": ["dom:read"]
    }
  ],
  "capabilities": ["dom:read"]
}
```

### 2. Implement Your Tools

Create tool implementations that will run in the browser sandbox:

```javascript
// tools/getPageContent.js
export default function getPageContent({ selector = 'main', format = 'text' }) {
  const element = document.querySelector(selector);

  if (!element) {
    throw new Error(`Element not found: ${selector}`);
  }

  let content;
  switch (format) {
    case 'html':
      content = element.innerHTML;
      break;
    case 'markdown':
      content = htmlToMarkdown(element.innerHTML);
      break;
    default:
      content = element.textContent;
  }

  return {
    content,
    wordCount: content.split(/\s+/).filter(Boolean).length
  };
}
```

### 3. Register with BTCP Client

Register your manifest with the page's BTCP client:

```javascript
import { BTCPClient } from '@btcp/client';

const client = new BTCPClient();

await client.registerManifest({
  manifest: myManifest,
  implementations: {
    getPageContent: getPageContentImpl
  }
});
```

## Manifest Structure

### Root Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `btcp` | string | Yes | Protocol version (e.g., "1.0") |
| `name` | string | Yes | Unique identifier for this manifest |
| `version` | string | Yes | Semantic version of the manifest |
| `description` | string | Yes | Human-readable description |
| `provider` | object | No | Information about the tool provider |
| `tools` | array | Yes | List of tool definitions |
| `capabilities` | array | Yes | Union of all tool capabilities |
| `config` | object | No | Runtime configuration options |

### Tool Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique tool identifier (alphanumeric, underscores) |
| `description` | string | Yes | AI-readable description of the tool |
| `inputSchema` | object | Yes | JSON Schema for input parameters |
| `outputSchema` | object | No | JSON Schema for return value |
| `capabilities` | array | Yes | Required capabilities for this tool |
| `examples` | array | No | Example invocations |
| `deprecated` | boolean | No | Whether the tool is deprecated |

### Provider Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Provider display name |
| `url` | string | No | Provider website |
| `contact` | string | No | Support contact email |

## Capabilities Reference

### DOM Capabilities

```json
{
  "capabilities": [
    "dom:read",      // Read page content
    "dom:write",     // Modify page content
    "dom:observe"    // Observe DOM changes
  ]
}
```

### Storage Capabilities

```json
{
  "capabilities": [
    "storage:local:read",     // Read localStorage
    "storage:local:write",    // Write localStorage
    "storage:session:read",   // Read sessionStorage
    "storage:session:write",  // Write sessionStorage
    "storage:indexed:read",   // Read IndexedDB
    "storage:indexed:write"   // Write IndexedDB
  ]
}
```

### Network Capabilities

```json
{
  "capabilities": [
    "network:fetch:same-origin",   // Same-origin requests
    "network:fetch:cross-origin",  // Cross-origin requests
    "network:websocket"            // WebSocket connections
  ]
}
```

### Browser Capabilities

```json
{
  "capabilities": [
    "clipboard:read",    // Read clipboard
    "clipboard:write",   // Write clipboard
    "media:camera",      // Camera access
    "media:microphone",  // Microphone access
    "geolocation"        // Location access
  ]
}
```

## Writing Effective Tool Descriptions

AI agents rely on your descriptions to understand when and how to use tools. Follow these guidelines:

### Do
- Be specific about what the tool does
- Describe expected inputs clearly
- Mention important constraints or limitations
- Include the context in which the tool is useful

### Don't
- Use vague terms like "handles" or "manages"
- Assume AI knows your application's terminology
- Omit important side effects
- Forget to document error conditions

### Example

**Poor description:**
```json
{
  "name": "updateData",
  "description": "Updates the data"
}
```

**Good description:**
```json
{
  "name": "updateSpreadsheetCell",
  "description": "Updates the value of a single cell in the currently active spreadsheet. The cell is identified by its A1-notation address (e.g., 'B5'). Supports text, numbers, and formulas (prefix with '='). Triggers automatic recalculation of dependent cells. Fails if the cell is protected or the sheet is locked."
}
```

## Input Validation

BTCP validates inputs against your JSON Schema before execution. Provide detailed schemas to catch errors early:

```json
{
  "inputSchema": {
    "type": "object",
    "properties": {
      "email": {
        "type": "string",
        "format": "email",
        "description": "User's email address"
      },
      "age": {
        "type": "integer",
        "minimum": 0,
        "maximum": 150,
        "description": "User's age in years"
      },
      "role": {
        "type": "string",
        "enum": ["admin", "user", "guest"],
        "default": "user",
        "description": "Permission level"
      }
    },
    "required": ["email"],
    "additionalProperties": false
  }
}
```

## Error Handling

Tools should throw descriptive errors that help AI agents recover:

```javascript
export default function deleteItem({ itemId }) {
  const item = document.querySelector(`[data-id="${itemId}"]`);

  if (!item) {
    throw new BTCPError('ITEM_NOT_FOUND', {
      message: `No item found with ID: ${itemId}`,
      suggestion: 'Use listItems tool to get valid item IDs'
    });
  }

  if (item.dataset.protected === 'true') {
    throw new BTCPError('PERMISSION_DENIED', {
      message: `Item ${itemId} is protected and cannot be deleted`,
      suggestion: 'Only unprotected items can be deleted'
    });
  }

  item.remove();
  return { success: true, deletedId: itemId };
}
```

## Examples

Provide examples to help AI agents understand usage patterns:

```json
{
  "name": "searchProducts",
  "description": "Search for products in the catalog",
  "inputSchema": { ... },
  "examples": [
    {
      "description": "Search for red shoes",
      "input": {
        "query": "shoes",
        "filters": { "color": "red" }
      },
      "output": {
        "results": [...],
        "totalCount": 42
      }
    },
    {
      "description": "Search with price range",
      "input": {
        "query": "laptop",
        "filters": {
          "priceMin": 500,
          "priceMax": 1000
        }
      }
    }
  ]
}
```

## Context-Aware Registration

Register tools dynamically based on the current page or application state:

```javascript
const client = new BTCPClient();

// Register different tools based on page type
if (document.querySelector('.spreadsheet-app')) {
  await client.registerManifest(spreadsheetManifest);
} else if (document.querySelector('.email-app')) {
  await client.registerManifest(emailManifest);
}

// Re-register when navigation occurs
window.addEventListener('popstate', async () => {
  await client.clearManifests();
  await registerContextualTools();
});
```

## Testing Your Tools

### Unit Testing

Test tool implementations in isolation:

```javascript
import { createMockDOM } from '@btcp/testing';
import getPageContent from './tools/getPageContent';

describe('getPageContent', () => {
  it('extracts text from main element', () => {
    const dom = createMockDOM('<main>Hello World</main>');
    const result = getPageContent({ format: 'text' }, { dom });

    expect(result.content).toBe('Hello World');
    expect(result.wordCount).toBe(2);
  });
});
```

### Integration Testing

Test with the full BTCP client:

```javascript
import { BTCPTestClient } from '@btcp/testing';

describe('Spreadsheet Tools', () => {
  let client;

  beforeEach(async () => {
    client = new BTCPTestClient();
    await client.registerManifest(spreadsheetManifest);
    await client.grantCapabilities(['dom:read', 'dom:write']);
  });

  it('formats cells correctly', async () => {
    const result = await client.callTool('formatCells', {
      range: 'A1:A10',
      format: 'currency'
    });

    expect(result.cellsAffected).toBe(10);
  });
});
```

## Best Practices

1. **Minimal Capabilities**: Request only the capabilities you need
2. **Idempotency**: Design tools to be safely re-runnable when possible
3. **Atomic Operations**: Prefer tools that complete fully or fail cleanly
4. **Clear Naming**: Use verb-noun patterns (e.g., `getUser`, `createPost`)
5. **Versioning**: Update manifest version when tool behavior changes
6. **Documentation**: Keep descriptions in sync with implementations

## Next Steps

- **[API Reference: Manifest](./api/core/manifest.md)**: Complete manifest schema
- **[API Reference: Tool](./api/core/tool.md)**: Complete tool schema
- **[Security Model](./security.md)**: Capability system details
- **[Sandbox Protocols](./protocols/index.md)**: Execution environments
