# Manifest Schema

The Manifest is the root document that describes a collection of BTCP tools. This document provides the complete schema specification.

## Overview

A Manifest contains:
- Protocol version and metadata
- Provider information
- Tool definitions
- Capability requirements
- Configuration options

## Schema

### JSON Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://btcp.dev/schemas/manifest.json",
  "title": "BTCP Manifest",
  "description": "Browser Tool Calling Protocol manifest schema",
  "type": "object",
  "required": ["btcp", "name", "version", "tools", "capabilities"],
  "properties": {
    "btcp": {
      "type": "string",
      "pattern": "^[0-9]+\\.[0-9]+$",
      "description": "BTCP protocol version (e.g., '1.0')"
    },
    "name": {
      "type": "string",
      "pattern": "^[a-z][a-z0-9-]*$",
      "minLength": 1,
      "maxLength": 64,
      "description": "Unique identifier for this manifest"
    },
    "version": {
      "type": "string",
      "pattern": "^[0-9]+\\.[0-9]+\\.[0-9]+",
      "description": "Semantic version of this manifest"
    },
    "description": {
      "type": "string",
      "maxLength": 500,
      "description": "Human-readable description of the tool collection"
    },
    "provider": {
      "$ref": "#/$defs/Provider"
    },
    "tools": {
      "type": "array",
      "items": {
        "$ref": "#/$defs/Tool"
      },
      "minItems": 1,
      "description": "List of tools provided by this manifest"
    },
    "capabilities": {
      "type": "array",
      "items": {
        "$ref": "#/$defs/Capability"
      },
      "description": "Union of all capabilities required by tools"
    },
    "config": {
      "$ref": "#/$defs/Config"
    }
  },
  "$defs": {
    "Provider": {
      "type": "object",
      "properties": {
        "name": {
          "type": "string",
          "maxLength": 100,
          "description": "Provider display name"
        },
        "url": {
          "type": "string",
          "format": "uri",
          "description": "Provider website URL"
        },
        "contact": {
          "type": "string",
          "format": "email",
          "description": "Support contact email"
        },
        "icon": {
          "type": "string",
          "format": "uri",
          "description": "URL to provider icon (PNG, 64x64)"
        }
      },
      "required": ["name"]
    },
    "Tool": {
      "$ref": "tool.json"
    },
    "Capability": {
      "type": "string",
      "pattern": "^[a-z]+:[a-z]+(:[a-z-]+)?$",
      "description": "Capability identifier"
    },
    "Config": {
      "type": "object",
      "properties": {
        "timeout": {
          "type": "integer",
          "minimum": 1000,
          "maximum": 300000,
          "default": 30000,
          "description": "Default tool execution timeout in milliseconds"
        },
        "sandbox": {
          "type": "string",
          "enum": ["worker", "iframe", "ses", "wasm"],
          "default": "worker",
          "description": "Preferred sandbox implementation"
        },
        "maxConcurrent": {
          "type": "integer",
          "minimum": 1,
          "maximum": 10,
          "default": 3,
          "description": "Maximum concurrent tool executions"
        }
      }
    }
  }
}
```

## Fields

### btcp

**Type:** `string`
**Required:** Yes
**Pattern:** `^[0-9]+\.[0-9]+$`

The BTCP protocol version this manifest conforms to.

```json
{
  "btcp": "1.0"
}
```

### name

**Type:** `string`
**Required:** Yes
**Pattern:** `^[a-z][a-z0-9-]*$`
**Length:** 1-64 characters

A unique identifier for this manifest. Must be lowercase alphanumeric with hyphens.

```json
{
  "name": "my-app-tools"
}
```

### version

**Type:** `string`
**Required:** Yes
**Pattern:** Semantic versioning

The version of this manifest following [Semantic Versioning](https://semver.org/).

```json
{
  "version": "1.2.3"
}
```

### description

**Type:** `string`
**Required:** No
**Length:** Max 500 characters

A human-readable description of the tool collection.

```json
{
  "description": "Tools for interacting with the Acme spreadsheet application"
}
```

### provider

**Type:** `Provider` object
**Required:** No

Information about the organization providing these tools.

```json
{
  "provider": {
    "name": "Acme Corp",
    "url": "https://acme.example.com",
    "contact": "support@acme.example.com",
    "icon": "https://acme.example.com/icon.png"
  }
}
```

### tools

**Type:** `Tool[]` array
**Required:** Yes
**Min Items:** 1

List of tools provided by this manifest. See [Tool Schema](./tool.md) for details.

### capabilities

**Type:** `Capability[]` array
**Required:** Yes

The union of all capabilities required by tools in this manifest. This allows BTCP clients to request all necessary permissions upfront.

```json
{
  "capabilities": [
    "dom:read",
    "dom:write",
    "storage:local:read"
  ]
}
```

### config

**Type:** `Config` object
**Required:** No

Runtime configuration options.

```json
{
  "config": {
    "timeout": 60000,
    "sandbox": "ses",
    "maxConcurrent": 5
  }
}
```

## Complete Example

```json
{
  "btcp": "1.0",
  "name": "spreadsheet-tools",
  "version": "2.1.0",
  "description": "Tools for interacting with web-based spreadsheet applications",
  "provider": {
    "name": "Acme Productivity",
    "url": "https://acme.example.com",
    "contact": "support@acme.example.com",
    "icon": "https://acme.example.com/btcp-icon.png"
  },
  "tools": [
    {
      "name": "getCellValue",
      "description": "Retrieves the value of a specific cell by its address",
      "inputSchema": {
        "type": "object",
        "properties": {
          "cell": {
            "type": "string",
            "pattern": "^[A-Z]+[0-9]+$",
            "description": "Cell address in A1 notation (e.g., 'B5')"
          }
        },
        "required": ["cell"]
      },
      "outputSchema": {
        "type": "object",
        "properties": {
          "value": {
            "oneOf": [
              { "type": "string" },
              { "type": "number" },
              { "type": "boolean" },
              { "type": "null" }
            ]
          },
          "formula": {
            "type": "string",
            "description": "Cell formula if present"
          },
          "formatted": {
            "type": "string",
            "description": "Display-formatted value"
          }
        },
        "required": ["value"]
      },
      "capabilities": ["dom:read"]
    },
    {
      "name": "setCellValue",
      "description": "Sets the value of a specific cell. Supports text, numbers, and formulas (prefix with '=')",
      "inputSchema": {
        "type": "object",
        "properties": {
          "cell": {
            "type": "string",
            "pattern": "^[A-Z]+[0-9]+$",
            "description": "Cell address in A1 notation"
          },
          "value": {
            "oneOf": [
              { "type": "string" },
              { "type": "number" },
              { "type": "boolean" }
            ],
            "description": "Value to set (use '=' prefix for formulas)"
          }
        },
        "required": ["cell", "value"]
      },
      "outputSchema": {
        "type": "object",
        "properties": {
          "success": { "type": "boolean" },
          "previousValue": {},
          "newValue": {}
        },
        "required": ["success"]
      },
      "capabilities": ["dom:read", "dom:write"]
    },
    {
      "name": "getSelectedRange",
      "description": "Returns information about the currently selected cell range",
      "inputSchema": {
        "type": "object",
        "properties": {}
      },
      "outputSchema": {
        "type": "object",
        "properties": {
          "range": {
            "type": "string",
            "description": "Range in A1:B2 notation"
          },
          "rows": { "type": "integer" },
          "columns": { "type": "integer" },
          "values": {
            "type": "array",
            "items": {
              "type": "array"
            }
          }
        },
        "required": ["range", "rows", "columns"]
      },
      "capabilities": ["dom:read"]
    }
  ],
  "capabilities": [
    "dom:read",
    "dom:write"
  ],
  "config": {
    "timeout": 30000,
    "sandbox": "worker",
    "maxConcurrent": 3
  }
}
```

## Validation

BTCP clients MUST validate manifests before registration:

1. Schema validation against JSON Schema
2. Capability consistency (tool capabilities subset of manifest capabilities)
3. Tool name uniqueness
4. Version format validation

```javascript
import { validateManifest } from '@btcp/validator';

const result = validateManifest(manifest);

if (!result.valid) {
  console.error('Invalid manifest:', result.errors);
}
```

## Versioning

When updating a manifest:

- **Patch version** (1.0.0 → 1.0.1): Bug fixes, documentation changes
- **Minor version** (1.0.0 → 1.1.0): New tools added, optional fields added
- **Major version** (1.0.0 → 2.0.0): Breaking changes, removed tools, changed schemas

## Related

- [Tool Schema](./tool.md)
- [Capability Reference](./capabilities.md)
- [For Tool Providers](../for-tool-providers.md)
