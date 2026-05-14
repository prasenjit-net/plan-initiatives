# MCP Platform HTTP API Reference

## Base URL
```
http://localhost:8080
```

## Authentication
- Optional: `Authorization: Bearer <token>` header
- Session: `X-Session-ID: <session-id>` header or `session_id` cookie

## Common Response Headers
```
X-Trace-ID: <correlation-id>
X-Request-ID: <request-id>
X-Cache: HIT | MISS          ← present on tool call responses
Content-Type: application/json
```

---

## Tool Naming Convention

All tools in the platform catalog are namespaced as `{service_id}::{tool_name}`.

- **Namespaced (always works):** `weather_api_mcp::getCurrentWeather`
- **Bare name (works only if globally unique):** `getCurrentWeather`
- **Ambiguous bare name → 409 Conflict:**
  ```json
  {
    "error": "ambiguous_tool_name",
    "message": "Tool name 'getCurrentWeather' matches multiple services",
    "candidates": [
      "weather_api_mcp::getCurrentWeather",
      "backup_weather_rest::getCurrentWeather"
    ]
  }
  ```

Use namespaced names in production agents to avoid ambiguity as services are added.

---

## Endpoints

### 1. Health Check

**GET /health**

Health check for load balancers. Returns 200 if router is running.

**Response (200 OK):**
```json
{
  "status": "ok",
  "timestamp": "2026-05-13T12:34:56Z"
}
```

---

### 2. Readiness Check

**GET /ready**

Readiness probe for Kubernetes. Returns 200 if all dependencies are ready.

**Response (200 OK):**
```json
{
  "status": "ready",
  "dependencies": {
    "redis": "ok",
    "registry": "ok",
    "tracer": "ok"
  }
}
```

**Response (503 Service Unavailable):**
```json
{
  "status": "not_ready",
  "dependencies": {
    "redis": "failed",
    "registry": "ok",
    "tracer": "ok"
  }
}
```

---

### 3. Initialize MCP

**POST /mcp/initialize**

Handshake to discover router capabilities.

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

**Response (200 OK):**
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
      "sse_transport": true,
      "response_caching": true
    }
  }
}
```

---

### 3.5. MCP SSE Transport

The MCP spec defines transport over HTTP+SSE. Agents using official MCP SDKs should use this transport. Stateless `POST /mcp/call` remains available for simple clients.

#### Establish SSE Stream

**GET /mcp**

Opens a persistent Server-Sent Events connection. The server pushes JSON-RPC responses back over this stream. The client sends requests via `POST /mcp`.

**Request Headers:**
```
Accept: text/event-stream
X-Session-ID: <optional session id>
```

**Response Headers (200 OK — connection stays open):**
```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no
```

**Initial endpoint event (sent immediately on connect):**
```
event: endpoint
data: {"uri":"/mcp","session_id":"sess_abc123"}

```

**Heartbeat event (sent every 30 seconds to keep connection alive):**
```
event: ping
data: {"type":"ping","timestamp":"2026-05-14T07:00:00Z"}

```

**Tool call response event:**
```
event: message
data: {"jsonrpc":"2.0","id":"req_001","result":{"type":"text","text":"SF: 22°C, partly cloudy"},"meta":{"session_id":"sess_abc123","trace_id":"550e8400-e29b-41d4-a716-446655440000","latency_ms":142}}

```

**Error event:**
```
event: message
data: {"jsonrpc":"2.0","id":"req_001","error":{"code":-32603,"message":"Service unavailable"}}

```

**Example curl:**
```bash
curl -N -H "Accept: text/event-stream" http://localhost:8080/mcp
```

---

#### Send Request via SSE Session

**POST /mcp**

Send a JSON-RPC request bound to an existing SSE session. Response arrives as an SSE event on the open `GET /mcp` stream.

**Headers:**
```
Content-Type: application/json
X-Session-ID: sess_abc123
```

**Request:**
```json
{
  "jsonrpc": "2.0",
  "id": "req_001",
  "method": "tools/call",
  "params": {
    "name": "weather_api_mcp::getCurrentWeather",
    "arguments": { "city": "San Francisco", "units": "celsius" }
  }
}
```

**Response (202 Accepted):**
```json
{ "status": "accepted", "id": "req_001" }
```
The actual result arrives as an SSE `message` event on the open stream.

---

**GET /mcp/resources**

List all available tools from all registered services.

**Query Parameters:**
- `service` (optional): Filter by service name
- `format` (optional): Response format (default: `json`)

**Response (200 OK):**
```json
{
  "tools": [
    {
      "name": "weather_api_mcp::getCurrentWeather",
      "description": "Get current weather for a city",
      "input_schema": {
        "type": "object",
        "properties": {
          "city": {
            "type": "string",
            "description": "City name"
          },
          "units": {
            "type": "string",
            "enum": ["celsius", "fahrenheit"],
            "default": "celsius"
          }
        },
        "required": ["city"]
      },
      "output_schema": {
        "type": "object",
        "properties": {
          "temperature": { "type": "number" },
          "condition": { "type": "string" }
        }
      }
    },
    {
      "name": "stock_api_rest::getPrice",
      "description": "Get current stock price",
      "input_schema": {
        "type": "object",
        "properties": {
          "symbol": { "type": "string" }
        },
        "required": ["symbol"]
      }
    }
  ],
  "count": 2,
  "meta": {
    "trace_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

---

### 5. List Available Prompts

**GET /mcp/prompts**

List all available prompts from registered services.

**Response (200 OK):**
```json
{
  "prompts": [
    {
      "name": "translate_to_spanish",
      "description": "Translate text to Spanish",
      "arguments": [
        {
          "name": "text",
          "description": "Text to translate",
          "required": true
        }
      ]
    }
  ],
  "count": 1,
  "meta": {
    "trace_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

---

### 6. Call a Tool

**POST /mcp/call**

Invoke a tool and get the result.

**Headers:**
```
X-Session-ID: <session-id>  (optional)
```

**Request:**
```json
{
  "jsonrpc": "2.0",
  "id": "call_001",
  "method": "tools/call",
  "params": {
    "name": "weather_api_mcp::getCurrentWeather",
    "arguments": {
      "city": "San Francisco",
      "units": "celsius"
    }
  }
}
```

**Response (200 OK):**
```json
{
  "jsonrpc": "2.0",
  "id": "call_001",
  "result": {
    "type": "text",
    "text": "San Francisco: 22°C, partly cloudy, wind 10 mph"
  },
  "meta": {
    "session_id": "sess_abc123",
    "trace_id": "550e8400-e29b-41d4-a716-446655440000",
    "latency_ms": 142,
    "service": "weather_api_mcp",
    "instance": "weather_mcp_1:9000",
    "cache": {
      "hit": false
    }
  }
}
```

**Response (404 Tool Not Found):**
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

**Response (429 Rate Limited):**
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

**Response (503 Service Unavailable):**
```json
{
  "jsonrpc": "2.0",
  "id": "call_001",
  "error": {
    "code": -32603,
    "message": "Service unavailable",
    "data": {
      "service": "weather_api_mcp",
      "reason": "no healthy instances"
    }
  }
}
```

---

### 7. Create Session

**POST /sessions**

Create a new session for tracking context across multiple tool calls.

**Request:**
```json
{
  "user_id": "agent_user_1",
  "ttl_seconds": 3600,
  "context": {
    "conversation_id": "conv_123",
    "language": "en",
    "custom_field": "custom_value"
  }
}
```

**Response (201 Created):**
```json
{
  "session_id": "sess_6789",
  "user_id": "agent_user_1",
  "created_at": "2026-05-13T12:34:56Z",
  "expires_at": "2026-05-13T13:34:56Z",
  "ttl_seconds": 3600,
  "context": {
    "conversation_id": "conv_123",
    "language": "en",
    "custom_field": "custom_value"
  },
  "meta": {
    "trace_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**Location Header:**
```
Location: /sessions/sess_6789
```

---

### 8. Get Session

**GET /sessions/{session_id}**

Retrieve session details and tool invocation history.

**Response (200 OK):**
```json
{
  "session_id": "sess_6789",
  "user_id": "agent_user_1",
  "created_at": "2026-05-13T12:34:56Z",
  "expires_at": "2026-05-13T13:34:56Z",
  "last_activity": "2026-05-13T12:36:20Z",
  "context": {
    "conversation_id": "conv_123",
    "language": "en",
    "custom_field": "custom_value"
  },
  "tool_invocations": [
    {
      "id": "inv_001",
      "tool": "weather_getCurrentWeather",
      "arguments": {
        "city": "San Francisco",
        "units": "celsius"
      },
      "result": "San Francisco: 22°C, partly cloudy",
      "timestamp": "2026-05-13T12:35:10Z",
      "latency_ms": 142,
      "trace_id": "550e8400-e29b-41d4-a716-446655440000"
    },
    {
      "id": "inv_002",
      "tool": "stock_getPrice",
      "arguments": {
        "symbol": "GOOGL"
      },
      "result": "GOOGL: $185.50 USD",
      "timestamp": "2026-05-13T12:36:20Z",
      "latency_ms": 87,
      "trace_id": "550e8400-e29b-41d4-a716-446655440000"
    }
  ],
  "tool_invocation_count": 2,
  "meta": {
    "trace_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**Response (404 Not Found):**
```json
{
  "error": "session_not_found",
  "message": "Session sess_invalid does not exist",
  "meta": {
    "trace_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**Response (410 Gone - Expired):**
```json
{
  "error": "session_expired",
  "message": "Session sess_6789 has expired",
  "meta": {
    "trace_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

---

### 9. Update Session

**PATCH /sessions/{session_id}**

Update session context or extend TTL.

**Request:**
```json
{
  "context": {
    "user_preference": "new_value"
  },
  "extend_ttl_seconds": 1800
}
```

**Response (200 OK):**
```json
{
  "session_id": "sess_6789",
  "expires_at": "2026-05-13T14:04:56Z",
  "context": {
    "conversation_id": "conv_123",
    "language": "en",
    "custom_field": "custom_value",
    "user_preference": "new_value"
  }
}
```

---

### 10. Delete Session

**DELETE /sessions/{session_id}**

Terminate a session and clean up associated data.

**Response (204 No Content):**
```
(empty body)
```

**Response (404 Not Found):**
```json
{
  "error": "session_not_found",
  "message": "Session sess_invalid does not exist"
}
```

---

### 11. List Services (Admin)

**GET /admin/services**

List all registered MCP servers and REST APIs (admin only).

**Headers:**
```
Authorization: Bearer <admin-token>
```

**Response (200 OK):**
```json
{
  "services": [
    {
      "id": "weather_api_mcp",
      "type": "mcp",
      "endpoint": "localhost:9000",
      "status": "healthy",
      "instances": 2,
      "healthy_instances": 2,
      "tools_count": 5,
      "last_health_check": "2026-05-13T12:34:50Z"
    },
    {
      "id": "stock_api_rest",
      "type": "rest",
      "endpoint": "https://api.stocks.com",
      "status": "healthy",
      "instances": 3,
      "healthy_instances": 2,
      "tools_count": 3,
      "last_health_check": "2026-05-13T12:34:48Z"
    }
  ],
  "count": 2,
  "meta": {
    "trace_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

---

### 12. Get Service Status (Admin)

**GET /admin/services/{service_id}**

Detailed status of a specific service.

**Response (200 OK):**
```json
{
  "id": "weather_api_mcp",
  "type": "mcp",
  "endpoint": "localhost:9000",
  "status": "healthy",
  "instances": [
    {
      "id": "weather_mcp_1",
      "endpoint": "localhost:9000",
      "status": "healthy",
      "latency_ms": 45,
      "last_health_check": "2026-05-13T12:34:50Z",
      "active_connections": 5
    },
    {
      "id": "weather_mcp_2",
      "endpoint": "localhost:9001",
      "status": "healthy",
      "latency_ms": 52,
      "last_health_check": "2026-05-13T12:34:50Z",
      "active_connections": 3
    }
  ],
  "tools": [
    {
      "name": "weather_getCurrentWeather",
      "invocation_count": 1234,
      "error_rate": 0.01,
      "avg_latency_ms": 48
    }
  ]
}
```

---

## Status Codes Summary

| Code | Meaning | When |
|------|---------|------|
| 200 | OK | Successful tool call, session retrieval |
| 201 | Created | Session created |
| 204 | No Content | Session deleted |
| 400 | Bad Request | Invalid input, schema validation failed |
| 401 | Unauthorized | Authentication failed |
| 404 | Not Found | Tool not found, session not found, unknown endpoint |
| 410 | Gone | Session expired |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Error | Unexpected condition |
| 502 | Bad Gateway | Backend service error |
| 503 | Service Unavailable | No healthy instances |
| 504 | Gateway Timeout | Backend took too long |

---

## Error Response Format

All error responses follow this format:

```json
{
  "error": "error_code",
  "message": "Human-readable error message",
  "meta": {
    "trace_id": "550e8400-e29b-41d4-a716-446655440000",
    "request_id": "req_123"
  },
  "details": {
    "field": "additional context"
  }
}
```

---

## Rate Limiting

Rate limiting is enforced per session. Response includes headers:

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1620000000
X-RateLimit-RetryAfter: 60
```

---

## Pagination

List endpoints support pagination:

**Query Parameters:**
- `limit` (default: 50, max: 500)
- `offset` (default: 0)

**Response:**
```json
{
  "items": [...],
  "count": 50,
  "total": 234,
  "limit": 50,
  "offset": 0,
  "has_more": true
}
```

---

## Request/Response Examples

### Example 1: Single Tool Invocation

```bash
curl -X POST http://localhost:8080/mcp/call \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "weather_getCurrentWeather",
      "arguments": {
        "city": "Paris"
      }
    }
  }'
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "type": "text",
    "text": "Paris: 18°C, rainy"
  },
  "meta": {
    "trace_id": "550e8400-e29b-41d4-a716-446655440000",
    "latency_ms": 156
  }
}
```

---

### Example 2: Multi-Turn Conversation with Session

```bash
# Step 1: Create session
curl -X POST http://localhost:8080/sessions \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user_42",
    "ttl_seconds": 3600
  }' | jq -r '.session_id'
# Output: sess_6789

# Step 2: First tool call with session
curl -X POST http://localhost:8080/mcp/call \
  -H "Content-Type: application/json" \
  -H "X-Session-ID: sess_6789" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "weather_getCurrentWeather",
      "arguments": { "city": "London" }
    }
  }'

# Step 3: Second tool call (same session)
curl -X POST http://localhost:8080/mcp/call \
  -H "Content-Type: application/json" \
  -H "X-Session-ID: sess_6789" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "weather_getCurrentWeather",
      "arguments": { "city": "Paris" }
    }
  }'

# Step 4: Retrieve session history
curl -X GET http://localhost:8080/sessions/sess_6789
```

---

**Next:** See [docs/DEPLOYMENT.md](DEPLOYMENT.md) for deployment and operational guidance.
