# Capability Reference

Capabilities are permissions that tools must declare and users must grant. This document provides the complete capability reference.

## Overview

BTCP uses a capability-based security model:

1. **Declaration**: Tools declare required capabilities in their schema
2. **Request**: BTCP client requests capabilities from user
3. **Grant**: User approves or denies capabilities
4. **Enforcement**: Sandbox enforces capability restrictions

## Capability Format

Capabilities follow a hierarchical namespace:

```
category:resource:action
```

Examples:
- `dom:read` - Read DOM (category:action)
- `storage:local:write` - Write localStorage (category:resource:action)
- `network:fetch:cross-origin` - Cross-origin fetch (category:action:modifier)

## DOM Capabilities

Access to page content and structure.

### dom:read

**Risk Level:** Low

Read access to DOM content and structure.

**Allows:**
- `document.querySelector()` / `querySelectorAll()`
- `element.textContent` / `innerHTML` (read)
- `element.getAttribute()`
- `getComputedStyle()`
- `element.getBoundingClientRect()`

**Example Use Cases:**
- Extracting page content
- Reading form values
- Analyzing page structure

```javascript
// Requires: dom:read
const title = document.querySelector('h1').textContent;
```

### dom:write

**Risk Level:** Medium

Modify DOM content and structure.

**Allows:**
- `element.textContent` / `innerHTML` (write)
- `element.setAttribute()`
- `element.classList.add()` / `remove()`
- `document.createElement()`
- `element.appendChild()` / `removeChild()`
- `element.click()` (synthetic events)

**Example Use Cases:**
- Filling forms
- Clicking buttons
- Modifying page content

```javascript
// Requires: dom:read, dom:write
document.querySelector('#email').value = 'user@example.com';
document.querySelector('form').submit();
```

### dom:observe

**Risk Level:** Low

Observe DOM mutations.

**Allows:**
- `MutationObserver`
- `ResizeObserver`
- `IntersectionObserver`

**Example Use Cases:**
- Waiting for dynamic content
- Monitoring page changes

```javascript
// Requires: dom:observe
const observer = new MutationObserver(callback);
observer.observe(element, { childList: true });
```

### dom:shadow

**Risk Level:** Medium

Access shadow DOM trees.

**Allows:**
- `element.shadowRoot`
- Querying within shadow roots

**Example Use Cases:**
- Interacting with web components
- Accessing encapsulated content

## Storage Capabilities

Access to browser storage APIs.

### storage:local:read

**Risk Level:** Low

Read from localStorage.

**Allows:**
- `localStorage.getItem()`
- `localStorage.key()`
- `localStorage.length`

### storage:local:write

**Risk Level:** Medium

Write to localStorage.

**Allows:**
- `localStorage.setItem()`
- `localStorage.removeItem()`
- `localStorage.clear()`

### storage:session:read

**Risk Level:** Low

Read from sessionStorage.

**Allows:**
- `sessionStorage.getItem()`
- `sessionStorage.key()`
- `sessionStorage.length`

### storage:session:write

**Risk Level:** Medium

Write to sessionStorage.

**Allows:**
- `sessionStorage.setItem()`
- `sessionStorage.removeItem()`
- `sessionStorage.clear()`

### storage:indexed:read

**Risk Level:** Medium

Read from IndexedDB.

**Allows:**
- Opening databases (read-only)
- Reading object stores
- Using indexes

### storage:indexed:write

**Risk Level:** High

Write to IndexedDB.

**Allows:**
- Creating/deleting databases
- Creating/modifying object stores
- Adding/updating/deleting records

### storage:cookie:read

**Risk Level:** High

Read cookies.

**Allows:**
- `document.cookie` (read)

**Security Note:** May expose session tokens. Use with caution.

### storage:cookie:write

**Risk Level:** Critical

Write cookies.

**Allows:**
- `document.cookie` (write)

**Security Note:** Can modify authentication state. Rarely needed.

## Network Capabilities

Access to network APIs.

### network:fetch:same-origin

**Risk Level:** Medium

Make same-origin HTTP requests.

**Allows:**
- `fetch()` to same origin
- `XMLHttpRequest` to same origin

**Restrictions:**
- URL must match page origin
- Standard CORS rules apply

```javascript
// Requires: network:fetch:same-origin
const response = await fetch('/api/data');
```

### network:fetch:cross-origin

**Risk Level:** High

Make cross-origin HTTP requests.

**Allows:**
- `fetch()` to any origin
- `XMLHttpRequest` to any origin

**Security Note:** Can exfiltrate data. Requires careful review.

```javascript
// Requires: network:fetch:cross-origin
const response = await fetch('https://api.external.com/data');
```

### network:websocket:same-origin

**Risk Level:** Medium

Open same-origin WebSocket connections.

**Allows:**
- `new WebSocket()` to same origin

### network:websocket:cross-origin

**Risk Level:** High

Open cross-origin WebSocket connections.

**Allows:**
- `new WebSocket()` to any origin

## Clipboard Capabilities

Access to clipboard APIs.

### clipboard:read

**Risk Level:** High

Read clipboard content.

**Allows:**
- `navigator.clipboard.readText()`
- `navigator.clipboard.read()`

**Security Note:** May expose sensitive copied data.

### clipboard:write

**Risk Level:** Medium

Write to clipboard.

**Allows:**
- `navigator.clipboard.writeText()`
- `navigator.clipboard.write()`

## Media Capabilities

Access to media devices.

### media:camera

**Risk Level:** Critical

Access camera.

**Allows:**
- `navigator.mediaDevices.getUserMedia({ video: true })`

**Security Note:** Privacy-sensitive. Requires strong justification.

### media:microphone

**Risk Level:** Critical

Access microphone.

**Allows:**
- `navigator.mediaDevices.getUserMedia({ audio: true })`

**Security Note:** Privacy-sensitive. Requires strong justification.

### media:screen

**Risk Level:** Critical

Capture screen content.

**Allows:**
- `navigator.mediaDevices.getDisplayMedia()`

**Security Note:** Can capture any visible content.

## Location Capabilities

### geolocation

**Risk Level:** High

Access user location.

**Allows:**
- `navigator.geolocation.getCurrentPosition()`
- `navigator.geolocation.watchPosition()`

## Notification Capabilities

### notifications

**Risk Level:** Low

Show notifications.

**Allows:**
- `new Notification()`
- `Notification.requestPermission()`

## Capability Combinations

Common capability patterns:

### Read-Only Web Scraping

```json
{
  "capabilities": ["dom:read"]
}
```

### Form Automation

```json
{
  "capabilities": ["dom:read", "dom:write"]
}
```

### Data Export

```json
{
  "capabilities": ["dom:read", "clipboard:write"]
}
```

### API Integration

```json
{
  "capabilities": ["dom:read", "network:fetch:same-origin", "storage:local:read"]
}
```

### Full Application Control

```json
{
  "capabilities": [
    "dom:read",
    "dom:write",
    "storage:local:read",
    "storage:local:write",
    "network:fetch:same-origin"
  ]
}
```

## Wildcard Capabilities

For development/testing only:

```json
{
  "capabilities": ["dom:*", "storage:*"]
}
```

**Warning:** Wildcard capabilities should never be used in production.

## Capability Inheritance

Some capabilities imply others:

| Capability | Implies |
|------------|---------|
| `dom:write` | `dom:read` |
| `storage:local:write` | `storage:local:read` |
| `storage:session:write` | `storage:session:read` |
| `storage:indexed:write` | `storage:indexed:read` |
| `network:fetch:cross-origin` | `network:fetch:same-origin` |

## Risk Levels

| Level | Description | User Experience |
|-------|-------------|-----------------|
| Low | Minimal risk | May auto-grant |
| Medium | Moderate risk | Requires consent |
| High | Significant risk | Requires explicit consent + warning |
| Critical | Severe risk | Requires explicit consent + strong warning |

## Related

- [Security Model](../security.md)
- [Tool Schema](./tool.md)
- [Manifest Schema](./manifest.md)
