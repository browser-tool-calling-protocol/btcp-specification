# Tool Schema

A Tool is a callable function exposed to AI agents. This document provides the complete schema specification.

## Overview

Each tool defines:
- A unique name and description
- Input parameters (JSON Schema)
- Output structure (JSON Schema)
- Required capabilities
- Optional examples and metadata

## Schema

### JSON Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://btcp.dev/schemas/tool.json",
  "title": "BTCP Tool",
  "description": "Browser Tool Calling Protocol tool schema",
  "type": "object",
  "required": ["name", "description", "inputSchema", "capabilities"],
  "properties": {
    "name": {
      "type": "string",
      "pattern": "^[a-zA-Z][a-zA-Z0-9_]*$",
      "minLength": 1,
      "maxLength": 64,
      "description": "Unique tool identifier"
    },
    "description": {
      "type": "string",
      "minLength": 10,
      "maxLength": 1000,
      "description": "AI-readable description of the tool"
    },
    "inputSchema": {
      "$ref": "https://json-schema.org/draft/2020-12/schema",
      "description": "JSON Schema for input parameters"
    },
    "outputSchema": {
      "$ref": "https://json-schema.org/draft/2020-12/schema",
      "description": "JSON Schema for return value"
    },
    "capabilities": {
      "type": "array",
      "items": {
        "type": "string",
        "pattern": "^[a-z]+:[a-z]+(:[a-z-]+)?$"
      },
      "description": "Required capabilities for this tool"
    },
    "examples": {
      "type": "array",
      "items": {
        "$ref": "#/$defs/Example"
      },
      "description": "Example invocations"
    },
    "deprecated": {
      "type": "boolean",
      "default": false,
      "description": "Whether the tool is deprecated"
    },
    "deprecationMessage": {
      "type": "string",
      "description": "Migration guidance for deprecated tools"
    },
    "tags": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "description": "Categorization tags"
    },
    "timeout": {
      "type": "integer",
      "minimum": 1000,
      "maximum": 300000,
      "description": "Tool-specific timeout override"
    }
  },
  "$defs": {
    "Example": {
      "type": "object",
      "properties": {
        "description": {
          "type": "string",
          "description": "What this example demonstrates"
        },
        "input": {
          "type": "object",
          "description": "Example input arguments"
        },
        "output": {
          "description": "Expected output"
        }
      },
      "required": ["input"]
    }
  }
}
```

## Fields

### name

**Type:** `string`
**Required:** Yes
**Pattern:** `^[a-zA-Z][a-zA-Z0-9_]*$`
**Length:** 1-64 characters

A unique identifier for the tool within the manifest. Must start with a letter and contain only alphanumeric characters and underscores.

**Naming Conventions:**
- Use camelCase: `getCellValue`, `formatRange`
- Use verb-noun pattern: `get`, `set`, `create`, `delete`, `list`, `update`
- Be specific: `searchProducts` not `search`

```json
{
  "name": "getCellValue"
}
```

### description

**Type:** `string`
**Required:** Yes
**Length:** 10-1000 characters

An AI-readable description of what the tool does. This is critical for tool selection by AI agents.

**Guidelines:**
- Start with an action verb
- Describe inputs and outputs
- Mention important constraints
- Include context about when to use

```json
{
  "description": "Retrieves the value and formula of a spreadsheet cell by its A1-notation address. Returns null for empty cells. Works only on the currently active sheet."
}
```

### inputSchema

**Type:** JSON Schema object
**Required:** Yes

Defines the expected input parameters using [JSON Schema](https://json-schema.org/).

```json
{
  "inputSchema": {
    "type": "object",
    "properties": {
      "cell": {
        "type": "string",
        "pattern": "^[A-Z]+[0-9]+$",
        "description": "Cell address in A1 notation (e.g., 'B5', 'AA100')"
      },
      "includeFormatting": {
        "type": "boolean",
        "default": false,
        "description": "Whether to include cell formatting information"
      }
    },
    "required": ["cell"],
    "additionalProperties": false
  }
}
```

**Best Practices:**
- Always include `description` for each property
- Use `required` to specify mandatory fields
- Set `additionalProperties: false` to catch typos
- Use appropriate `format` values (email, uri, date-time)
- Provide `default` values where sensible
- Use `enum` for fixed option sets

### outputSchema

**Type:** JSON Schema object
**Required:** No (but recommended)

Defines the structure of the return value.

```json
{
  "outputSchema": {
    "type": "object",
    "properties": {
      "value": {
        "oneOf": [
          { "type": "string" },
          { "type": "number" },
          { "type": "boolean" },
          { "type": "null" }
        ],
        "description": "The cell's raw value"
      },
      "formula": {
        "type": "string",
        "description": "The cell's formula, if any"
      },
      "displayValue": {
        "type": "string",
        "description": "The formatted display value"
      }
    },
    "required": ["value"]
  }
}
```

### capabilities

**Type:** `string[]` array
**Required:** Yes

List of capabilities required to execute this tool.

```json
{
  "capabilities": ["dom:read", "storage:local:read"]
}
```

**Standard Capabilities:**

| Category | Capabilities |
|----------|--------------|
| DOM | `dom:read`, `dom:write`, `dom:observe` |
| Storage | `storage:local:read`, `storage:local:write`, `storage:session:read`, `storage:session:write` |
| Network | `network:fetch:same-origin`, `network:fetch:cross-origin` |
| Clipboard | `clipboard:read`, `clipboard:write` |

### examples

**Type:** `Example[]` array
**Required:** No

Example invocations to help AI agents understand usage patterns.

```json
{
  "examples": [
    {
      "description": "Get value of cell B5",
      "input": {
        "cell": "B5"
      },
      "output": {
        "value": 42,
        "formula": null,
        "displayValue": "42"
      }
    },
    {
      "description": "Get a formula cell",
      "input": {
        "cell": "C10"
      },
      "output": {
        "value": 150,
        "formula": "=SUM(A1:A10)",
        "displayValue": "$150.00"
      }
    }
  ]
}
```

### deprecated

**Type:** `boolean`
**Required:** No
**Default:** `false`

Marks the tool as deprecated.

```json
{
  "deprecated": true,
  "deprecationMessage": "Use getCellValues (plural) instead for better performance"
}
```

### tags

**Type:** `string[]` array
**Required:** No

Categorization tags for tool discovery.

```json
{
  "tags": ["spreadsheet", "cells", "read"]
}
```

### timeout

**Type:** `integer`
**Required:** No
**Range:** 1000-300000 (1s to 5min)

Tool-specific timeout override in milliseconds.

```json
{
  "timeout": 60000
}
```

## Complete Examples

### Read-Only Tool

```json
{
  "name": "getPageTitle",
  "description": "Returns the title of the current page from the document's <title> element",
  "inputSchema": {
    "type": "object",
    "properties": {},
    "additionalProperties": false
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "title": {
        "type": "string",
        "description": "The page title"
      }
    },
    "required": ["title"]
  },
  "capabilities": ["dom:read"],
  "tags": ["page", "metadata"]
}
```

### Write Tool

```json
{
  "name": "fillFormField",
  "description": "Fills a form field identified by name, id, or label text with the specified value. Works with input, textarea, and select elements.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "identifier": {
        "type": "string",
        "description": "Field name, id, or label text"
      },
      "value": {
        "type": "string",
        "description": "Value to fill"
      },
      "identifierType": {
        "type": "string",
        "enum": ["name", "id", "label", "auto"],
        "default": "auto",
        "description": "How to interpret the identifier"
      }
    },
    "required": ["identifier", "value"],
    "additionalProperties": false
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "success": {
        "type": "boolean"
      },
      "fieldType": {
        "type": "string",
        "enum": ["input", "textarea", "select"]
      },
      "previousValue": {
        "type": "string"
      }
    },
    "required": ["success"]
  },
  "capabilities": ["dom:read", "dom:write"],
  "examples": [
    {
      "description": "Fill email field by name",
      "input": {
        "identifier": "email",
        "value": "user@example.com"
      },
      "output": {
        "success": true,
        "fieldType": "input",
        "previousValue": ""
      }
    }
  ],
  "tags": ["forms", "input"]
}
```

### Complex Tool with Multiple Capabilities

```json
{
  "name": "exportTableToClipboard",
  "description": "Exports a table element to the clipboard in the specified format. The table is identified by a CSS selector. Supports CSV, TSV, and JSON formats.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "selector": {
        "type": "string",
        "description": "CSS selector for the table element"
      },
      "format": {
        "type": "string",
        "enum": ["csv", "tsv", "json"],
        "default": "csv",
        "description": "Export format"
      },
      "includeHeaders": {
        "type": "boolean",
        "default": true,
        "description": "Whether to include table headers"
      }
    },
    "required": ["selector"],
    "additionalProperties": false
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "success": { "type": "boolean" },
      "rowCount": { "type": "integer" },
      "columnCount": { "type": "integer" },
      "byteSize": { "type": "integer" }
    },
    "required": ["success", "rowCount", "columnCount"]
  },
  "capabilities": ["dom:read", "clipboard:write"],
  "timeout": 10000,
  "examples": [
    {
      "description": "Export main data table as CSV",
      "input": {
        "selector": "#data-table",
        "format": "csv"
      },
      "output": {
        "success": true,
        "rowCount": 50,
        "columnCount": 5,
        "byteSize": 2048
      }
    }
  ],
  "tags": ["export", "clipboard", "tables"]
}
```

## Implementation Guidelines

### Input Handling

```javascript
export default function getCellValue({ cell, includeFormatting = false }) {
  // Inputs are already validated against inputSchema
  // 'cell' is guaranteed to match pattern ^[A-Z]+[0-9]+$

  const element = findCellElement(cell);

  if (!element) {
    throw new BTCPError('CELL_NOT_FOUND', {
      message: `Cell ${cell} not found in the active sheet`,
      suggestion: 'Verify the cell address and ensure a sheet is active'
    });
  }

  const result = {
    value: extractValue(element),
    formula: extractFormula(element),
    displayValue: element.textContent
  };

  if (includeFormatting) {
    result.formatting = extractFormatting(element);
  }

  return result;
}
```

### Error Responses

When a tool fails, throw a `BTCPError` with:
- Error code (UPPER_SNAKE_CASE)
- Human-readable message
- Optional suggestion for recovery

```javascript
throw new BTCPError('PERMISSION_DENIED', {
  message: 'Cannot modify protected cells',
  suggestion: 'Unprotect the sheet or select unprotected cells'
});
```

## Related

- [Manifest Schema](./manifest.md)
- [Capability Reference](./capabilities.md)
- [For Tool Providers](../for-tool-providers.md)
