# MCP Platform - Protocol Specification

## Table of Contents
1. [Overview](#overview)
2. [Transport Layer](#transport-layer)
3. [MCP Protocol Basics](#mcp-protocol-basics)
4. [Router-Specific Protocol Extensions](#router-specific-protocol-extensions)
5. [Message Format](#message-format)
6. [Handshake & Session Management](#handshake--session-management)
7. [Error Handling](#error-handling)
8. [Examples](#examples)

---

## Overview

The MCP Platform implements the Model Context Protocol (MCP) specification with minor extensions to support distributed routing, session management, and protocol translation.

**Key Points:**
- Implements MCP 1.0 spec (from Anthropic)
- Supports **two transports**: HTTP+SSE (recommended) and stateless HTTP
- Maintains backward compatibility with standard MCP servers
- Adds session, tracing, and tool namespacing extensions

---

## Transport Layer

### Transport 1: HTTP+SSE (Recommended)

The MCP specification defines HTTP+SSE as the standard transport. Agents using official MCP SDKs should use this transport.

```
Client                                   MCP Platform
  │                                           │
  │  GET /mcp                                 │
  │  Accept: text/event-stream                │
  ├──────────────────────────────────────────►│
  │                                           │  (connection stays open)
  │◄──── event: endpoint ─────────────────────┤
  │      data: {"uri":"/mcp",                 │
  │             "session_id":"sess_abc"}       │
  │                                           │
  │  POST /mcp                                │  (client sends request)
  │  X-Session-ID: sess_abc                   │
  │  body: {"jsonrpc":"2.0","id":"1",         │
  │          "method":"tools/call",...}        │
  ├──────────────────────────────────────────►│
  │                                           │
  │◄──── 202 Accepted ────────────────────────┤  (immediate ack)
  │                                           │
  │◄──── event: message ──────────────────────┤  (result via SSE stream)
  │      data: {"jsonrpc":"2.0","id":"1",     │
  │             "result":{...}}               │
  │                                           │
  │◄──── event: ping (every 30s) ─────────────┤  (keepalive)
```

### Transport 2: Stateless HTTP

For simple clients, scripts, or legacy integrations that do not support SSE.

```
Client                                   MCP Platform
  │                                           │
  │  POST /mcp/call                           │
  │  Content-Type: application/json           │
  │  body: {"jsonrpc":"2.0","id":"1",...}     │
  ├──────────────────────────────────────────►│
  │                                           │
  │◄──── 200 OK ──────────────────────────────┤
  │      body: {"jsonrpc":"2.0","id":"1",     │
  │             "result":{...}}               │
```

Both transports share the same middleware pipeline, router, and adapter stack. Transport only affects request delivery and response streaming.

**Configuration:** `MCP_TRANSPORT=sse|http` (default: `sse`)

---

## MCP Protocol Basics

### Standard MCP Message Structure

```json
{
  "jsonrpc": "2.0",
  "id": "unique-request-id",
  "method": "method-name",
  "params": {
    "param1": "value1",
    "param2": "value2"
  }
}
```

### Core MCP Methods

| Method | Purpose | From | To |
|--------|---------|------|-----|
| `initialize` | Handshake | Client → Server | Exchange capabilities |
| `tools/list` | List available tools | Client → Server | Returns tool definitions |
| `tools/call` | Invoke a tool | Client → Server | Returns tool result |
| `resources/list` | List available resources | Client → Server | Returns resource definitions |
| `resources/read` | Read a resource | Client → Server | Returns resource content |
| `prompts/list` | List available prompts | Client → Server | Returns prompt definitions |
| `prompts/get` | Get a prompt | Client → Server | Returns prompt |

---

## Router-Specific Protocol Extensions

### Tool Name Namespacing

All tools exposed in the platform catalog use the format `{service_id}::{tool_name}`. This prevents collisions when multiple services register tools with identical names.

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "weather_api_mcp::getCurrentWeather",
    "arguments": { "city": "London" }
  }
}
```

Agents may use a bare name (e.g., `getCurrentWeather`) only when it is globally unique. If ambiguous, the platform returns:
```json
{
  "error": {
    "code": -32049,
    "message": "Ambiguous tool name; use namespaced form",
    "data": {
      "candidates": [
        "weather_api_mcp::getCurrentWeather",
        "backup_weather_rest::getCurrentWeather"
      ]
    }
  }
}
```

### Session Management

The MCP Platform extends MCP to track sessions across multiple tool calls.

#### Session Metadata
```json
{
  "session_id": "uuid-string",
  "trace_id": "uuid-string (optional)",
  "correlation_id": "string (optional)",
  "user_id": "string (optional)"
}
```

#### Session Propagation
Sessions are propagated via:
1. **HTTP Header:** `X-Session-ID: <session-id>`
2. **HTTP Cookie:** `session_id=<session-id>`
3. **MCP Parameter:** Included in `params.session` (optional)

### Tracing & Observability

Request-level tracing metadata:

```json
{
  "trace_id": "550e8400-e29b-41d4-a716-446655440000",
  "span_id": "f9d9f9d9f9d9f9d9",
  "parent_span_id": "a1b2c3d4e5f6a1b2 (optional)"
}
```

Metadata is included in:
- HTTP headers: `X-Trace-ID`, `X-Span-ID`
- Response JSON: `meta.trace_id`, `meta.span_id`
- Logs: All log entries

---

## Message Format

### Request Format (HTTP POST)

```http
POST /mcp/call HTTP/1.1
Host: router.example.com
Content-Type: application/json
X-Session-ID: session_abc123
X-Trace-ID: 550e8400-e29b-41d4-a716-446655440000

{
  "jsonrpc": "2.0",
  "id": "req_001",
  "method": "tools/call",
  "params": {
    "name": "weather_getCurrentWeather",
    "arguments": {
      "city": "San Francisco",
      "units": "celsius"
    }
  }
}
```

### Response Format (HTTP 200 - Success)

```http
HTTP/1.1 200 OK
Content-Type: application/json
X-Trace-ID: 550e8400-e29b-41d4-a716-446655440000

{
  "jsonrpc": "2.0",
  "id": "req_001",
  "result": {
    "type": "text",
    "text": "San Francisco: 22°C, partially cloudy"
  },
  "meta": {
    "trace_id": "550e8400-e29b-41d4-a716-446655440000",
    "session_id": "session_abc123",
    "latency_ms": 142,
    "service": "weather_api",
    "instance": "weather_server_1:9000"
  }
}
```

### Response Format (Error - 4xx/5xx)

```http
HTTP/1.1 502 Bad Gateway
Content-Type: application/json
X-Trace-ID: 550e8400-e29b-41d4-a716-446655440000

{
  "jsonrpc": "2.0",
  "id": "req_001",
  "error": {
    "code": -32603,
    "message": "Internal error",
    "data": {
      "error": "backend_service_error",
      "service": "weather_api",
      "instance": "weather_server_1:9000",
      "backend_error": "connection timeout"
    }
  },
  "meta": {
    "trace_id": "550e8400-e29b-41d4-a716-446655440000",
    "session_id": "session_abc123",
    "latency_ms": 10001
  }
}
```

---

## Handshake & Session Management

### 1. Initialize (Discover Capabilities)

**Request:**
```json
{
  "jsonrpc": "2.0",
  "id": "init_001",
  "method": "initialize",
  "params": {
    "client_name": "agent_v1",
    "client_version": "1.0.0",
    "capabilities": {
      "tools": true,
      "resources": true,
      "prompts": true
    }
  }
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "id": "init_001",
  "result": {
    "server_name": "mcp-platform",
    "server_version": "1.0.0",
    "capabilities": {
      "tools": true,
      "resources": true,
      "prompts": true,
      "session_management": true,
      "tracing": true,
      "rate_limiting": true
    }
  }
}
```

### 2. Create Session

**Request:**
```http
POST /sessions HTTP/1.1
Content-Type: application/json

{
  "user_id": "user_123",
  "ttl_seconds": 3600,
  "context": {
    "conversation_id": "conv_456",
    "language": "en"
  }
}
```

**Response:**
```json
{
  "session_id": "sess_6789",
  "expires_at": "2026-05-13T13:34:56Z",
  "created_at": "2026-05-13T12:34:56Z",
  "ttl_seconds": 3600
}
```

### 3. Tool Invocation with Session

**Request (include session ID in header):**
```http
POST /mcp/call HTTP/1.1
Content-Type: application/json
X-Session-ID: sess_6789

{
  "jsonrpc": "2.0",
  "id": "call_001",
  "method": "tools/call",
  "params": {
    "name": "weather_getCurrentWeather",
    "arguments": { "city": "London" }
  }
}
```

**Response (session ID echoed back):**
```json
{
  "jsonrpc": "2.0",
  "id": "call_001",
  "result": { ... },
  "meta": {
    "session_id": "sess_6789",
    "trace_id": "...",
    ...
  }
}
```

### 4. Retrieve Session Context

**Request:**
```http
GET /sessions/sess_6789 HTTP/1.1
```

**Response:**
```json
{
  "session_id": "sess_6789",
  "user_id": "user_123",
  "created_at": "2026-05-13T12:34:56Z",
  "expires_at": "2026-05-13T13:34:56Z",
  "last_activity": "2026-05-13T12:35:10Z",
  "context": {
    "conversation_id": "conv_456",
    "language": "en"
  },
  "tool_invocations": [
    {
      "tool": "weather_getCurrentWeather",
      "arguments": { "city": "London" },
      "result": "...",
      "timestamp": "2026-05-13T12:35:10Z"
    }
  ]
}
```

---

## Error Handling

### JSON-RPC Error Codes

| Code | Meaning | HTTP Status |
|------|---------|-------------|
| -32700 | Parse error | 400 |
| -32600 | Invalid Request | 400 |
| -32601 | Method not found | 404 |
| -32602 | Invalid params | 400 |
| -32603 | Internal error | 500 |
| -32000 to -32099 | Server error (reserved) | 500 |

### Router-Specific Error Codes

| Code | Meaning | HTTP Status |
|------|---------|-------------|
| -32050 | Tool not found | 404 |
| -32051 | Service unavailable | 503 |
| -32052 | Rate limit exceeded | 429 |
| -32053 | Session not found | 440 |
| -32054 | Session expired | 440 |
| -32055 | Authentication failed | 401 |
| -32056 | Gateway timeout | 504 |

### Error Response Examples

**Tool not found:**
```json
{
  "jsonrpc": "2.0",
  "id": "call_001",
  "error": {
    "code": -32050,
    "message": "Tool not found",
    "data": {
      "tool_name": "unknown_tool"
    }
  }
}
```

**Rate limit exceeded:**
```json
{
  "jsonrpc": "2.0",
  "id": "call_001",
  "error": {
    "code": -32052,
    "message": "Rate limit exceeded",
    "data": {
      "retry_after": 5,
      "reset_at": "2026-05-13T12:35:10Z"
    }
  }
}
```

**Backend service error:**
```json
{
  "jsonrpc": "2.0",
  "id": "call_001",
  "error": {
    "code": -32603,
    "message": "Internal error",
    "data": {
      "service": "weather_api",
      "instance": "weather_1:9000",
      "backend_error": "connection refused"
    }
  }
}
```

---

## Examples

### Example 1: Complete Workflow

#### Step 1: Initialize
```bash
curl -X POST http://localhost:8080/mcp/initialize \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "client_name": "agent",
      "client_version": "1.0"
    }
  }'
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "server_name": "mcp-platform",
    "server_version": "1.0.0",
    "capabilities": {
      "tools": true,
      "resources": true,
      "prompts": true
    }
  }
}
```

#### Step 2: Create Session
```bash
curl -X POST http://localhost:8080/sessions \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "agent_user_1",
    "ttl_seconds": 3600
  }'
```

Response:
```json
{
  "session_id": "sess_7a9c2e",
  "expires_at": "2026-05-13T13:34:56Z",
  "created_at": "2026-05-13T12:34:56Z"
}
```

#### Step 3: List Available Tools
```bash
curl -X GET http://localhost:8080/mcp/resources \
  -H "X-Session-ID: sess_7a9c2e"
```

Response:
```json
{
  "tools": [
    {
      "name": "weather_getCurrentWeather",
      "description": "Get current weather for a city",
      "input_schema": {
        "type": "object",
        "properties": {
          "city": { "type": "string" },
          "units": { "type": "string", "enum": ["celsius", "fahrenheit"] }
        },
        "required": ["city"]
      }
    },
    {
      "name": "stock_getPrice",
      "description": "Get stock price",
      "input_schema": {
        "type": "object",
        "properties": {
          "symbol": { "type": "string" }
        },
        "required": ["symbol"]
      }
    }
  ]
}
```

#### Step 4: Call a Tool
```bash
curl -X POST http://localhost:8080/mcp/call \
  -H "Content-Type: application/json" \
  -H "X-Session-ID: sess_7a9c2e" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "weather_getCurrentWeather",
      "arguments": {
        "city": "San Francisco",
        "units": "celsius"
      }
    }
  }'
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "type": "text",
    "text": "San Francisco: 22°C, partly cloudy, wind 10 mph"
  },
  "meta": {
    "session_id": "sess_7a9c2e",
    "trace_id": "550e8400-e29b-41d4-a716-446655440000",
    "latency_ms": 145,
    "service": "weather_api_mcp",
    "instance": "weather_mcp_1:9000"
  }
}
```

#### Step 5: Call Another Tool (Same Session)
```bash
curl -X POST http://localhost:8080/mcp/call \
  -H "Content-Type: application/json" \
  -H "X-Session-ID: sess_7a9c2e" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "stock_getPrice",
      "arguments": {
        "symbol": "GOOGL"
      }
    }
  }'
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "type": "text",
    "text": "GOOGL: $185.50 USD"
  },
  "meta": {
    "session_id": "sess_7a9c2e",
    "trace_id": "550e8400-e29b-41d4-a716-446655440000",
    "latency_ms": 87,
    "service": "stock_api_rest",
    "instance": "stock_api:8888"
  }
}
```

#### Step 6: Retrieve Session to See History
```bash
curl -X GET http://localhost:8080/sessions/sess_7a9c2e
```

Response:
```json
{
  "session_id": "sess_7a9c2e",
  "user_id": "agent_user_1",
  "created_at": "2026-05-13T12:34:56Z",
  "expires_at": "2026-05-13T13:34:56Z",
  "last_activity": "2026-05-13T12:36:20Z",
  "tool_invocations": [
    {
      "tool": "weather_getCurrentWeather",
      "arguments": { "city": "San Francisco", "units": "celsius" },
      "result": "San Francisco: 22°C, partly cloudy, wind 10 mph",
      "timestamp": "2026-05-13T12:35:10Z",
      "latency_ms": 145
    },
    {
      "tool": "stock_getPrice",
      "arguments": { "symbol": "GOOGL" },
      "result": "GOOGL: $185.50 USD",
      "timestamp": "2026-05-13T12:36:20Z",
      "latency_ms": 87
    }
  ]
}
```

---

### Example 2: Error Scenarios

#### Scenario: Tool Not Found
```bash
curl -X POST http://localhost:8080/mcp/call \
  -H "Content-Type: application/json" \
  -H "X-Session-ID: sess_7a9c2e" \
  -d '{
    "jsonrpc": "2.0",
    "id": 4,
    "method": "tools/call",
    "params": {
      "name": "unknown_tool",
      "arguments": {}
    }
  }'
```

Response (HTTP 404):
```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "error": {
    "code": -32050,
    "message": "Tool not found",
    "data": {
      "tool_name": "unknown_tool",
      "available_tools": [
        "weather_getCurrentWeather",
        "stock_getPrice"
      ]
    }
  },
  "meta": {
    "trace_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

#### Scenario: Service Unavailable
```bash
curl -X POST http://localhost:8080/mcp/call \
  -H "Content-Type: application/json" \
  -H "X-Session-ID: sess_7a9c2e" \
  -d '{
    "jsonrpc": "2.0",
    "id": 5,
    "method": "tools/call",
    "params": {
      "name": "weather_getCurrentWeather",
      "arguments": { "city": "Tokyo" }
    }
  }'
```

Response (HTTP 503):
```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "error": {
    "code": -32603,
    "message": "Service unavailable",
    "data": {
      "service": "weather_api_mcp",
      "reason": "no healthy instances",
      "retry_after": 30
    }
  },
  "meta": {
    "trace_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

#### Scenario: Rate Limited
```bash
curl -X POST http://localhost:8080/mcp/call \
  -H "Content-Type: application/json" \
  -H "X-Session-ID: sess_7a9c2e" \
  -d '...'
```

Response (HTTP 429):
```json
{
  "error": {
    "code": -32052,
    "message": "Rate limit exceeded",
    "data": {
      "limit": 100,
      "window_seconds": 60,
      "retry_after": 45,
      "reset_at": "2026-05-13T12:37:00Z"
    }
  }
}
```

---

**Next:** See [docs/API.md](API.md) for HTTP API reference.
