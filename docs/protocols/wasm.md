# WebAssembly Sandbox

The WebAssembly (Wasm) sandbox executes BTCP tools in a hardware-isolated memory environment using WebAssembly.

## Overview

WebAssembly provides the strongest isolation guarantees through linear memory isolation and explicit import/export boundaries. This is ideal for compute-intensive tools or scenarios requiring maximum security.

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         Browser                                   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    JavaScript Host                          │  │
│  │                                                              │  │
│  │  ┌─────────────┐    ┌─────────────┐    ┌────────────────┐  │  │
│  │  │ BTCP Client │    │ Wasm Loader │    │ Import Bridge  │  │  │
│  │  │             │───►│             │───►│                │  │  │
│  │  │             │    │             │    │ dom_read()     │  │  │
│  │  │             │    │             │    │ storage_get()  │  │  │
│  │  └─────────────┘    └─────────────┘    └───────┬────────┘  │  │
│  │                                                 │           │  │
│  │  ┌──────────────────────────────────────────────▼────────┐ │  │
│  │  │                 WebAssembly Instance                   │ │  │
│  │  │                                                        │ │  │
│  │  │  ┌────────────────────────────────────────────────┐   │ │  │
│  │  │  │              Linear Memory                      │   │ │  │
│  │  │  │  ┌──────────────────────────────────────────┐  │   │ │  │
│  │  │  │  │              Tool Code                    │  │   │ │  │
│  │  │  │  │                                          │  │   │ │  │
│  │  │  │  │  fn execute(args) -> result {            │  │   │ │  │
│  │  │  │  │    let data = dom_read(selector);        │  │   │ │  │
│  │  │  │  │    return process(data);                 │  │   │ │  │
│  │  │  │  │  }                                       │  │   │ │  │
│  │  │  │  └──────────────────────────────────────────┘  │   │ │  │
│  │  │  └────────────────────────────────────────────────┘   │ │  │
│  │  │                                                        │ │  │
│  │  │  Exports: execute(), alloc(), dealloc()               │ │  │
│  │  └────────────────────────────────────────────────────────┘ │  │
│  │                                                              │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

## Isolation Properties

### Linear Memory Isolation
- Each Wasm instance has its own memory space
- Cannot access JavaScript heap directly
- No pointer escapes possible

### Explicit Boundaries
- All interactions through imports/exports
- Type-checked at module boundary
- No ambient access to browser APIs

### Deterministic Execution
- Same inputs produce same outputs
- No undefined behavior
- Predictable performance

## Implementation

### Wasm Runtime

```javascript
class WasmManager {
  constructor() {
    this.modules = new Map();
    this.instances = new Map();
  }

  async loadToolModule(toolId, wasmBytes) {
    const module = await WebAssembly.compile(wasmBytes);
    this.modules.set(toolId, module);
    return module;
  }

  async executeTool(toolId, args, capabilities) {
    const module = this.modules.get(toolId);
    if (!module) {
      throw new Error(`Tool module not loaded: ${toolId}`);
    }

    // Create capability-gated imports
    const imports = this.createImports(capabilities);

    // Instantiate module with imports
    const instance = await WebAssembly.instantiate(module, imports);

    // Allocate and write arguments to Wasm memory
    const argsPtr = this.writeToMemory(instance, JSON.stringify(args));

    try {
      // Call tool's execute function
      const resultPtr = instance.exports.execute(argsPtr);

      // Read result from Wasm memory
      const result = this.readFromMemory(instance, resultPtr);

      return JSON.parse(result);

    } finally {
      // Free allocated memory
      instance.exports.dealloc(argsPtr);
    }
  }

  createImports(capabilities) {
    const imports = {
      env: {
        // Memory management
        memory: new WebAssembly.Memory({ initial: 256 }),

        // Abort handler
        abort: (msg, file, line, column) => {
          throw new Error(`Wasm abort: ${msg} at ${file}:${line}:${column}`);
        }
      },
      btcp: {}
    };

    // Add capability-gated imports
    if (capabilities.includes('dom:read')) {
      imports.btcp.dom_query = this.createDomQueryImport();
      imports.btcp.dom_text = this.createDomTextImport();
    }

    if (capabilities.includes('dom:write')) {
      imports.btcp.dom_set_text = this.createDomSetTextImport();
      imports.btcp.dom_click = this.createDomClickImport();
    }

    if (capabilities.includes('storage:local:read')) {
      imports.btcp.storage_get = this.createStorageGetImport();
    }

    // Block ungranted capabilities
    const allImports = ['dom_query', 'dom_text', 'dom_set_text', 'dom_click', 'storage_get'];
    for (const imp of allImports) {
      if (!imports.btcp[imp]) {
        imports.btcp[imp] = () => {
          throw new BTCPCapabilityError(`Capability not granted for: ${imp}`);
        };
      }
    }

    return imports;
  }

  createDomQueryImport() {
    return (selectorPtr, selectorLen) => {
      const selector = this.decodeString(selectorPtr, selectorLen);
      const element = document.querySelector(selector);
      return element ? 1 : 0;
    };
  }

  createDomTextImport() {
    return (selectorPtr, selectorLen, resultPtr) => {
      const selector = this.decodeString(selectorPtr, selectorLen);
      const element = document.querySelector(selector);
      const text = element?.textContent ?? '';
      return this.writeString(resultPtr, text);
    };
  }
}
```

### Memory Management

```javascript
class WasmMemory {
  constructor(instance) {
    this.memory = instance.exports.memory;
    this.alloc = instance.exports.alloc;
    this.dealloc = instance.exports.dealloc;
  }

  writeString(str) {
    const encoder = new TextEncoder();
    const bytes = encoder.encode(str);

    // Allocate memory in Wasm
    const ptr = this.alloc(bytes.length + 4);

    // Write length prefix
    const view = new DataView(this.memory.buffer);
    view.setUint32(ptr, bytes.length, true);

    // Write string data
    const target = new Uint8Array(this.memory.buffer, ptr + 4, bytes.length);
    target.set(bytes);

    return ptr;
  }

  readString(ptr) {
    const view = new DataView(this.memory.buffer);
    const length = view.getUint32(ptr, true);

    const bytes = new Uint8Array(this.memory.buffer, ptr + 4, length);
    const decoder = new TextDecoder();

    return decoder.decode(bytes);
  }
}
```

### Tool Development (Rust Example)

```rust
// tool.rs
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    // BTCP imports
    #[wasm_bindgen(js_namespace = btcp)]
    fn dom_text(selector_ptr: *const u8, selector_len: usize, result_ptr: *mut u8) -> usize;

    #[wasm_bindgen(js_namespace = btcp)]
    fn dom_set_text(selector_ptr: *const u8, selector_len: usize, value_ptr: *const u8, value_len: usize) -> bool;
}

#[wasm_bindgen]
pub fn execute(args_ptr: *const u8) -> *mut u8 {
    // Parse arguments
    let args = parse_args(args_ptr);

    // Use BTCP APIs
    let title = get_dom_text("h1");

    // Build result
    let result = format!(r#"{{"title": "{}"}}"#, title);

    // Return result pointer
    string_to_ptr(result)
}

fn get_dom_text(selector: &str) -> String {
    let mut result = vec![0u8; 1024];

    unsafe {
        let len = dom_text(
            selector.as_ptr(),
            selector.len(),
            result.as_mut_ptr()
        );

        result.truncate(len);
        String::from_utf8_unchecked(result)
    }
}

// Memory allocation exports
#[wasm_bindgen]
pub fn alloc(size: usize) -> *mut u8 {
    let mut buf = Vec::with_capacity(size);
    let ptr = buf.as_mut_ptr();
    std::mem::forget(buf);
    ptr
}

#[wasm_bindgen]
pub fn dealloc(ptr: *mut u8, size: usize) {
    unsafe {
        let _ = Vec::from_raw_parts(ptr, 0, size);
    }
}
```

### Building Wasm Tools

```bash
# Using Rust and wasm-pack
wasm-pack build --target web

# Using AssemblyScript
asc tool.ts -o tool.wasm --runtime stub
```

## Configuration

```json
{
  "config": {
    "sandbox": "wasm",
    "wasmOptions": {
      "memoryInitial": 256,
      "memoryMaximum": 1024,
      "timeout": 30000
    }
  }
}
```

## Supported Languages

| Language | Toolchain | Notes |
|----------|-----------|-------|
| Rust | wasm-pack, wasm-bindgen | Best ecosystem |
| AssemblyScript | asc | TypeScript-like |
| C/C++ | Emscripten | Mature toolchain |
| Go | TinyGo | Smaller binaries |
| Zig | Built-in | Low-level control |

## Security Benefits

| Threat | Mitigation |
|--------|------------|
| Memory corruption | Linear memory isolation |
| Arbitrary code exec | No dynamic code generation |
| API abuse | Explicit imports only |
| Denial of service | Resource limits |
| Side channels | Constant-time operations |

## Performance Characteristics

| Aspect | Performance |
|--------|-------------|
| Compilation | ~10-100ms (cached) |
| Instantiation | ~1-10ms |
| Execution | Near-native |
| Memory overhead | Configurable |

## Limitations

| Limitation | Workaround |
|------------|------------|
| No direct DOM access | Import bridge |
| String handling overhead | Shared memory |
| Module size | Code splitting |
| Compilation time | Module caching |

## Browser Support

| Browser | Support |
|---------|---------|
| Chrome | ✅ Full |
| Firefox | ✅ Full |
| Safari | ✅ Full |
| Edge | ✅ Full |

All modern browsers support WebAssembly since 2017.

## Related

- [Protocols Overview](./index.md)
- [SES Sandbox](./ses.md)
- [Web Worker Sandbox](./worker.md)
- [Security Model](../security.md)
