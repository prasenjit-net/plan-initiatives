# MCP Platform Architecture

## Table of Contents
1. [System Architecture](#system-architecture)
2. [Component Design](#component-design)
3. [Data Flow](#data-flow)
4. [Concurrency Model](#concurrency-model)
5. [Error Handling](#error-handling)
6. [Scalability Considerations](#scalability-considerations)

> **v1.1 Additions:** SSE Transport, Circuit Breaker, Tool Name Namespacing, Response Cache, gRPC Adapter

---

## System Architecture

### Overview

The MCP Platform operates as a stateless message broker between AI agents and a heterogeneous collection of backend services (MCP servers, REST APIs, and gRPC services). Its primary responsibility is to aggregate, normalize, route, and monitor tool invocations.

### Layered Architecture

```
                AI Agent (MCP Client)
                        │
          ┌─────────────┴──────────────┐
          │  SSE Transport (preferred)  │  Stateless HTTP (simple clients)
          │  GET /mcp  ←──events        │  POST /mcp/call ──► response
          │  POST /mcp ──► requests     │
          └─────────────┬──────────────┘
                        │
┌───────────────────────▼──────────────────────────────────────────────┐
│                    HTTP Server + SSE Transport Layer                  │
│  GET /mcp (SSE) | POST /mcp | POST /mcp/call | GET /mcp/resources   │
└────────────────────────────────────────┬─────────────────────────────┘
                                         │
┌────────────────────────────────────────▼─────────────────────────────┐
│                      Middleware Pipeline                              │
├──────────────────────────────────────────────────────────────────────┤
│  1. Request Logger    (log request metadata, trace ID)               │
│  2. Tracer            (create OpenTelemetry spans)                   │
│  3. Session Extractor (load context from Redis)                      │
│  4. Rate Limiter      (check token bucket)                           │
│  5. Validator         (sanitize inputs, validate schema)             │
└────────────────────────────────────────┬─────────────────────────────┘
                                         │
┌────────────────────────────────────────▼─────────────────────────────┐
│                         Core Router Logic                             │
├──────────────────────────────────────────────────────────────────────┤
│  • Tool namespace resolution  (service_id::tool_name lookup)         │
│  • Service lookup in Registry                                        │
│  • Response Cache check       (return cached result if HIT)         │
│  • Select healthy instance    (Load Balancer)                        │
│  • Circuit Breaker check      (fail-fast if OPEN)                   │
│  • Delegate to adapter        (MCP, REST, or gRPC)                  │
│  • Cache result on miss       (if tool is cache-enabled)            │
└────────────────────────────────────────┬─────────────────────────────┘
                                         │
         ┌───────────────┬───────────────┼─────────────────┐
         │               │               │                 │
┌────────▼───────┐ ┌─────▼──────┐ ┌─────▼──────┐  ┌──────────────┐
│  MCP Adapter   │ │REST Adapter│ │gRPC Adapter│  │ Logging /    │
│ • MCP protocol │ │• REST→JSON │ │• Reflection│  │ Tracing      │
│ • Server calls │ │• Schema    │ │• Proto map │  │ • OTel spans │
│ • Reconnection │ │• HTTP exec │ │• Channel   │  │ • Metrics    │
└────────┬───────┘ └─────┬──────┘ └─────┬──────┘  └──────────────┘
         │               │               │
┌────────▼───────┐ ┌─────▼──────┐ ┌─────▼──────────────────────┐
│  MCP Servers   │ │  REST APIs │ │  gRPC Services             │
│  ├─ Server 1   │ │  ├─ Svc A  │ │  ├─ ServiceA:9090          │
│  ├─ Server 2   │ │  └─ Svc B  │ │  └─ ServiceB:9091          │
│  └─ Server N   │ └────────────┘ └────────────────────────────┘
└────────────────┘

External Services (shown separately):
  • Redis         (Session persistence + Response Cache [multi-instance])
  • Jaeger/OTel   (Distributed tracing)
  • Metrics Store (Prometheus/Datadog)
```

---

## Component Design

### 1. HTTP Server (`src/server/server.go`)

**Responsibility:** Accept HTTP requests, manage connection lifecycle, provide standard endpoints.

**Key Design Decisions:**
- **Stateless:** Router itself maintains no state; all state in Redis
- **Graceful Shutdown:** Accept no new requests, drain in-flight requests before exit
- **TLS Optional:** Support HTTPS for production deployments
- **Dual Transport:** Supports both SSE streaming transport and stateless HTTP transport
- **Health Endpoints:**
  - `GET /health` → 200 if responding (used by load balancers)
  - `GET /ready` → 200 if dependencies ready (used for startup probes)

**Endpoints:**
```
GET  /mcp            → SSE stream (MCP protocol over Server-Sent Events, preferred)
POST /mcp            → Single JSON-RPC request via SSE session (bound to SSE connection)
POST /mcp/call       → Stateless JSON-RPC tool invocation (simple clients)
GET  /mcp/resources  → List all available namespaced tools
GET  /mcp/prompts    → List all available prompts

POST /sessions       → Create session
GET  /sessions/{id}  → Retrieve session
DELETE /sessions/{id} → Delete session

GET  /health         → Liveness probe
GET  /ready          → Readiness probe (checks Redis, registry)
GET  /admin/services → List services (admin)
GET  /metrics        → Prometheus metrics
```

---

### 1.5. SSE Transport (`src/transport/sse.go`)

**Responsibility:** Manage persistent Server-Sent Events connections for MCP clients, providing the standard MCP transport as defined in the MCP specification.

**Why SSE Transport:**
The MCP specification defines transport over HTTP+SSE (not plain REST). Agents using official MCP SDKs (e.g., Anthropic's Python/TypeScript SDKs) expect:
- A persistent `GET /mcp` endpoint for server→client event streaming
- A `POST /mcp` endpoint for client→server requests, bound to the SSE session

Stateless `POST /mcp/call` remains available for simple scripts and legacy clients.

**Connection Lifecycle:**
```
1. Client opens: GET /mcp
   Headers: Accept: text/event-stream
            X-Session-ID: <optional>

2. Server responds with SSE headers:
   Content-Type: text/event-stream
   Cache-Control: no-cache
   Connection: keep-alive
   X-Accel-Buffering: no    ← disables nginx buffering

3. Server sends initial endpoint event:
   event: endpoint
   data: {"uri":"/mcp","session_id":"sess_abc123"}

4. Client sends requests to: POST /mcp
   Header: X-Session-ID: sess_abc123

5. Server streams responses back as SSE events:
   event: message
   data: {"jsonrpc":"2.0","id":"req_1","result":{...}}

6. Heartbeat every 30s (keeps connection alive through proxies):
   event: ping
   data: {"type":"ping","timestamp":"2026-05-14T07:00:00Z"}

7. Client disconnects or server closes on graceful shutdown.
```

**Concurrent Connection Limits:**
- Max active SSE connections: configurable via `MCP_SSE_MAX_CONNECTIONS` (default: 1000)
- Connections are tracked in a registry; excess connections receive HTTP 503
- Each connection spawns a lightweight goroutine for heartbeat/event delivery

**SSE Event Format:**
```
event: message
data: {"jsonrpc":"2.0","id":"req_001","result":{"type":"text","text":"SF: 22°C"}}

event: ping
data: {"type":"ping","timestamp":"2026-05-14T07:00:00Z"}

event: error
data: {"jsonrpc":"2.0","id":"req_001","error":{"code":-32603,"message":"Service unavailable"}}
```

**Shared Stack:** Both SSE and stateless HTTP transports pass through the same middleware pipeline (logging, tracing, session, rate limiting) and router core. Transport only changes the delivery mechanism, not the routing logic.

---

### 2. Service Registry (`src/registry/registry.go`)

**Responsibility:** Maintain a live inventory of available MCP servers, REST APIs, and gRPC services; track their capabilities, health, and namespaced tools.

**Tool Name Namespacing:**

All tools in the catalog use the format `{service_id}::{tool_name}` (e.g., `weather_mcp::getCurrentWeather`, `stock_rest::getPrice`). This eliminates collisions when two services expose identically-named tools.

- Agents may use the **bare name** (e.g., `getCurrentWeather`) only when it is globally unique across all services.
- If a bare name is **ambiguous** (matches tools in multiple services), the registry returns an error listing the candidate namespaced names.
- The router always resolves to a fully-qualified namespaced name before dispatching.

**Data Structure:**
```go
type ServiceType string

const (
  ServiceTypeMCP  ServiceType = "mcp"
  ServiceTypeREST ServiceType = "rest"
  ServiceTypeGRPC ServiceType = "grpc"   // NEW
)

type Service struct {
  ID              string                     // unique identifier (used as namespace prefix)
  Type            ServiceType                // mcp, rest, or grpc
  Endpoint        string                     // host:port or URL
  Tools           map[string]ToolDefinition  // bare tool name → definition
  HealthStatus    HealthStatus               // HEALTHY, UNHEALTHY
  InstanceCount   int
  LastHealthCheck time.Time
  LoadBalancer    *LoadBalancer
  CircuitBreaker  *CircuitBreaker            // NEW: per-service circuit breaker
}

type ToolDefinition struct {
  Name          string       // bare name within service (e.g., "getCurrentWeather")
  Namespace     string       // full namespaced name (e.g., "weather_mcp::getCurrentWeather")
  Description   string
  InputSchema   JSONSchema
  OutputSchema  JSONSchema
  CacheTTL      time.Duration // 0 = no caching; >0 = cache tool results for this duration
}
```

**Operations:**
- `RegisterService(service Service)` — Add or update a service; auto-assigns namespaces to all tools
- `DeregisterService(id string)` — Remove a service and its namespaced tools
- `GetServiceByToolName(name string) (Service, error)` — Accepts namespaced (`svc::tool`) or bare (`tool`) names; returns error with candidates if bare name is ambiguous
- `ListAllTools() []ToolDefinition` — Enumerate all tools with fully-qualified namespace names
- `GetServiceHealth(id string) HealthStatus`

**Concurrency:** Thread-safe with read-write locks (frequent reads, infrequent writes).

---

### 3. Router Core (`src/router/router.go`)

**Responsibility:** Orchestrate request flow: resolve namespace → check cache → select instance → check circuit breaker → delegate to adapter.

**Algorithm:**

```
Input: tool_call (tool_name, arguments, session_id)

1. Namespace resolution: full_name = registry.ResolveToolName(tool_name)
   → Accepts "weather_mcp::getCurrentWeather" or bare "getCurrentWeather"
   → If bare name is ambiguous: return 409 with candidate list
   → If not found: return 404

2. Registry lookup: service = registry.GetServiceByToolName(full_name)

3. Cache check (if tool has CacheTTL > 0):
   cache_key = "cache:" + full_name + ":" + sha256(json(arguments))
   if cached_result = cache.Get(cache_key):
     → return cached_result with X-Cache: HIT header

4. Load balancing: instance = service.LoadBalancer.SelectInstance()
   → If no healthy instances: return 503

5. Circuit breaker check: service.CircuitBreaker.Allow(instance)
   → If state is OPEN: return 503 with "reason: circuit_open", "retry_after: 30"

6. Adapter delegation: result = adapter.Call(service.Type, instance, tool_call)
   → On success: CircuitBreaker.RecordSuccess(instance)
   → On error: CircuitBreaker.RecordFailure(instance); try next instance

7. Cache store (on success, if CacheTTL > 0):
   cache.Set(cache_key, result, tool.CacheTTL)

8. Session update: session.ToolInvocations.append(invocation)
   session_store.UpdateSession(session)

9. Return: { "result": result, "session_id": session_id, "trace_id": trace_id,
             "cache": { "hit": false } }
```

**Error Handling:**
- Timeout: Return 504 Gateway Timeout if backend doesn't respond within deadline
- Unavailable: Return 503 Service Unavailable if no healthy instances
- Circuit Open: Return 503 with `"reason": "circuit_open"` and `"retry_after": 30`
- Ambiguous Tool: Return 409 Conflict with candidate namespaced names
- Invalid Tool: Return 404 Tool Not Found
- Invalid Arguments: Return 400 Bad Request with schema validation details
- Rate Limited: Return 429 Too Many Requests (handled by middleware)

---

### 4. MCP Adapter (`src/adapters/mcp/client.go`)

**Responsibility:** Establish and maintain connections to remote MCP servers, execute tool calls via MCP protocol.

**Connection Management:**
- Maintain connection pool (one connection per MCP server instance)
- Implement keep-alive (periodic pings)
- Auto-reconnect with exponential backoff (1s, 2s, 4s, 8s, max 60s)
- Timeout on all network operations (configurable, default 10s)

**Message Flow:**
```
Router → MCP Adapter:
  toolCall { name: "weather", args: { city: "SF" } }
  
MCP Adapter → MCP Server:
  {
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "weather",
      "arguments": { "city": "SF" }
    },
    "id": "uuid"
  }

MCP Server → MCP Adapter:
  {
    "jsonrpc": "2.0",
    "result": {
      "toolResult": {
        "type": "text",
        "text": "SF: 72°F, cloudy"
      }
    },
    "id": "uuid"
  }

MCP Adapter → Router:
  toolResult { value: "SF: 72°F, cloudy" }
```

**Error Handling:**
- Connection refused → Retry with backoff, mark service unhealthy
- Invalid JSON → Log error, mark service unhealthy
- MCP error response → Pass through to router
- Timeout → Retry, if all retries fail mark service unhealthy

---

### 5. REST Adapter (`src/adapters/rest/client.go` & `converter.go`)

**Responsibility:** Convert REST API schemas to MCP tool definitions, execute REST calls when tools are invoked.

**Schema Import (REST → MCP):**
```
Input: REST API spec (OpenAPI 3.0 or JSON Schema)

For each endpoint in spec:
  1. Convert endpoint to MCP tool definition:
     - tool name = operation ID (e.g., "weather_getCurrentWeather")
     - description = operation summary
     - input_schema = request parameters (query, body)
     - output_schema = response schema

  2. Generate tool metadata:
     - http_method = GET/POST/PUT/DELETE
     - url_pattern = /weather/{city}?units=celsius
     - auth_type = API_KEY | OAUTH | MTLS | NONE
     - content_type = application/json

Output: ToolDefinition[] compatible with MCP
```

**Execution (MCP Tool Call → REST → MCP Result):**
```
Input: mcp_tool_call { name: "weather_getCurrentWeather", args: { city: "SF", units: "celsius" } }

1. Lookup REST metadata: metadata = rest_registry.Get("weather_getCurrentWeather")

2. Build HTTP request:
   - URL: interpolate url_pattern with arguments
     e.g., /weather/SF?units=celsius
   - Headers: add auth (API key, OAuth token, mTLS cert)
   - Body: serialize arguments to JSON (if POST/PUT)
   - Timeout: 10s

3. Execute HTTP call: response, err = http.Do(request)

4. Convert response:
   - If error: convert HTTP error to MCP error
   - If success: map response body to MCP tool result

Output: mcp_tool_result { value: { ... } }
```

**Supported Auth Methods:**
- **API Key:** Header or query parameter
- **OAuth 2.0:** Bearer token (assume caller provides token)
- **mTLS:** Client certificate (loaded from file)
- **None:** Public endpoints

---

### 5.5. gRPC Adapter (`src/adapters/grpc/client.go`)

**Responsibility:** Invoke backend services exposed via gRPC, presenting them as MCP tools.

**Service Discovery:**
gRPC services are registered with the platform via config (`GRPC_SERVERS`). The adapter discovers available methods using the [gRPC Server Reflection API](https://grpc.github.io/grpc/core/md_doc_server-reflection-tutorial.html), requiring no manual protobuf files. Alternatively, a protobuf descriptor file can be provided for services that don't expose reflection.

**Tool Name Mapping:**
- Full namespaced tool name: `{service_id}::{ServiceName}.{MethodName}`
- Example: `payment_grpc::PaymentService.ChargeCard`
- Tool `input_schema` is auto-derived from the gRPC request message fields
- Tool `output_schema` is auto-derived from the gRPC response message fields

**Connection Management:**
```go
type GRPCAdapter struct {
  channels map[string]*grpc.ClientConn // one persistent channel per endpoint
  opts     []grpc.DialOption
}

// Channel options:
// - keepalive: ping every 30s, timeout 10s
// - TLS: configurable (insecure for dev, TLS/mTLS for prod)
// - backoff: exponential reconnect (1s → 60s)
// - max message size: 16MB
```

**Message Flow:**
```
Router → gRPC Adapter:
  toolCall { name: "payment_grpc::PaymentService.ChargeCard",
             args: { "card_token": "tok_abc", "amount_cents": 5000 } }

gRPC Adapter:
  1. Resolve channel: conn = channels["payment-svc:9090"]
  2. Use reflection to get method descriptor for PaymentService.ChargeCard
  3. Map JSON args → proto message (via protojson)
  4. Invoke: response, err = conn.Invoke(ctx, "/PaymentService/ChargeCard", req, resp)
  5. Map proto response → JSON

gRPC Adapter → Router:
  toolResult { value: { "transaction_id": "txn_xyz", "status": "approved" } }
```

**Auth Methods:**
- **Insecure (dev):** No TLS, no metadata auth
- **TLS:** Standard TLS with CA certificate verification
- **mTLS:** Mutual TLS with client certificate
- **Token metadata:** Bearer token injected as gRPC metadata header `authorization: Bearer <token>`

**gRPC Status → MCP/HTTP Error Mapping:**

| gRPC Status | HTTP Status | MCP Error Code |
|-------------|------------|----------------|
| OK | 200 | — |
| NOT_FOUND | 404 | -32050 |
| INVALID_ARGUMENT | 400 | -32602 |
| UNAVAILABLE | 503 | -32603 |
| DEADLINE_EXCEEDED | 504 | -32000 |
| PERMISSION_DENIED | 403 | -32051 |
| INTERNAL | 500 | -32603 |

---

**Responsibility:** Distribute tool calls across multiple instances of same service.

**Instance Selection Algorithm:**
```
// Round-robin (simple, fair distribution)
func (lb *LoadBalancer) SelectInstance() Instance {
  healthy := lb.FilterHealthyInstances()
  if len(healthy) == 0 {
    return nil, ErrNoHealthyInstances
  }
  instance := healthy[lb.roundRobinIndex % len(healthy)]
  lb.roundRobinIndex++
  return instance
}

// Least connections (optimal for long-running operations)
func (lb *LoadBalancer) SelectInstance() Instance {
  healthy := lb.FilterHealthyInstances()
  sort.By(healthy, func(i, j) Instance {
    return i.ActiveConnections < j.ActiveConnections
  })
  return healthy[0]
}
```

**Configuration:**
- Strategy: ROUND_ROBIN (default) or LEAST_CONNECTIONS
- Configurable per service

**Retry Logic:**
```
Max retries = number of healthy instances

for attempt := 0; attempt < maxRetries; attempt++ {
  instance := lb.SelectInstance()
  result, err := executeOnInstance(instance)
  if err == nil {
    return result
  }
  // retry with different instance
}
return error "all instances failed"
```

---

### 6.5. Circuit Breaker (`src/circuitbreaker/breaker.go`)

**Responsibility:** Prevent cascading failures by detecting degraded backends and failing-fast, giving them time to recover.

**States:**
```
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │   CLOSED ──(5 failures / >50% error rate)──► OPEN       │
  │      ▲                                        │          │
  │      │                                  (30s timeout)   │
  │      │                                        │          │
  │   SUCCESS ◄──── HALF-OPEN ◄─────────────────┘          │
  │      or                                                  │
  │   FAILURE ────────────────────────────────► OPEN        │
  └──────────────────────────────────────────────────────────┘
```

- **CLOSED:** Normal operation. Requests flow through. Failures are counted.
- **OPEN:** Backend is considered failing. All requests **fail immediately** with HTTP 503 (no network call made). After a 30-second cooldown, transitions to HALF-OPEN.
- **HALF-OPEN:** One probe request is allowed through. If it succeeds → CLOSED. If it fails → back to OPEN.

**State Transition Thresholds:**
- CLOSED → OPEN: 5 consecutive failures **or** >50% error rate in a 10-second rolling window
- OPEN → HALF-OPEN: 30-second cooldown (configurable via `CIRCUIT_BREAKER_TIMEOUT_SECONDS`)
- HALF-OPEN → CLOSED: 1 successful probe
- HALF-OPEN → OPEN: 1 failed probe

**Data Structure:**
```go
type CircuitBreakerState string

const (
  StateClosed   CircuitBreakerState = "closed"
  StateOpen     CircuitBreakerState = "open"
  StateHalfOpen CircuitBreakerState = "half_open"
)

type CircuitBreaker struct {
  state            CircuitBreakerState
  failureCount     int
  lastFailureTime  time.Time
  openUntil        time.Time
  mu               sync.RWMutex
  // config
  failureThreshold int           // default: 5
  timeout          time.Duration // default: 30s
  errorRateWindow  time.Duration // default: 10s
  errorRateLimit   float64       // default: 0.5 (50%)
}

func (cb *CircuitBreaker) Allow() bool        // returns false if OPEN
func (cb *CircuitBreaker) RecordSuccess()
func (cb *CircuitBreaker) RecordFailure()
func (cb *CircuitBreaker) State() CircuitBreakerState
```

**Error Response when OPEN:**
```json
{
  "error": "circuit_open",
  "message": "Service weather_mcp is temporarily unavailable",
  "retry_after": 30,
  "meta": { "trace_id": "..." }
}
```
HTTP 503 with header: `Retry-After: 30`

**Integration Point:** One `CircuitBreaker` instance is held per service (on the `Service` struct). The router checks `cb.Allow()` before every adapter call and calls `RecordSuccess()` or `RecordFailure()` after.

**Library:** Use [`sony/gobreaker`](https://github.com/sony/gobreaker) or a lightweight custom implementation.

---

**Responsibility:** Periodically verify that services are responsive; update registry.

**Health Check Algorithm:**
```
for each service in registry {
  for each instance of service {
    // MCP: Send MCP ping/initialize
    // REST: Send HEAD or GET to health endpoint
    
    request := BuildHealthCheckRequest(instance)
    response, latency, err := DoWithTimeout(request, 5s)
    
    if err != nil || response.StatusCode != 200 {
      instance.FailureCount++
      if instance.FailureCount > FAILURE_THRESHOLD {
        instance.Status = UNHEALTHY
      }
    } else {
      instance.FailureCount = 0
      instance.Status = HEALTHY
      instance.Latency = latency
    }
  }
}
```

**Configuration:**
- Check interval: 30 seconds (default)
- Failure threshold: 3 consecutive failures
- Check timeout: 5 seconds

---

### 8. Session Store (`src/session/store.go`)

**Responsibility:** Persist user/conversation context in Redis.

**Data Structure:**
```go
type Session struct {
  ID              string                    // UUID
  UserID          string                    // Optional
  CreatedAt       time.Time
  ExpiresAt       time.Time
  LastActivityAt  time.Time
  Context         map[string]interface{}    // Custom context
  ToolInvocations []ToolInvocation
}

type ToolInvocation struct {
  Tool       string
  Arguments  map[string]interface{}
  Result     map[string]interface{}
  Timestamp  time.Time
  Latency    time.Duration
}
```

**Redis Schema:**
```
Key: session:{session_id}
Value: JSON-serialized Session
TTL: ExpiresAt - now

Key: session_index:{user_id}
Value: Set of session IDs (for user-to-sessions lookup)
TTL: ExpiresAt - now
```

**Operations:**
- `CreateSession(userID string) Session` — Generate UUID, set TTL (24h default)
- `GetSession(sessionID string) (Session, error)` — Retrieve from Redis
- `UpdateSession(session Session) error` — Update in Redis, refresh TTL
- `DeleteSession(sessionID string) error` — Remove from Redis
- `ListSessionsByUserID(userID string) []Session` — Query by user

---

### 8.5. Response Cache (`src/cache/cache.go`)

**Responsibility:** Cache tool call results to reduce backend load for read-only, deterministic tools (e.g., weather lookups, stock prices, static data).

**Opt-in Model:** Caching is disabled by default. Tools opt in via `CacheTTL` in their `ToolDefinition`. Example: `CacheTTL: 60 * time.Second` caches results for 60 seconds.

**Cache Key:**
```
cache:{namespaced_tool_name}:{sha256(canonical_json(arguments))}

Example:
  cache:weather_mcp::getCurrentWeather:a3f5b9c2d1e4f7890123456789abcdef
```

**Backends:**
- **In-memory (`sync.Map` + TTL heap):** Single router instance. Fast, zero external deps. Evicted on restart.
- **Redis-backed:** Multi-instance router deployments. Consistent cache across all replicas. Configured via `CACHE_BACKEND=redis`.

**Flow:**
```
On tool call (tool has CacheTTL > 0):

1. Compute cache_key
2. result = cache.Get(cache_key)
   → HIT:  return result
           set X-Cache: HIT header
           include meta: { "cache": { "hit": true, "cached_at": "...", "ttl_remaining_s": 45 } }

3. → MISS: execute tool call normally
           cache.Set(cache_key, result, tool.CacheTTL)
           set X-Cache: MISS header
           include meta: { "cache": { "hit": false } }
```

**Data Structure:**
```go
type Cache interface {
  Get(key string) ([]byte, bool)
  Set(key string, value []byte, ttl time.Duration)
  Delete(key string)
  Flush() // clear all entries (admin)
}

// In-memory implementation: sync.Map + background TTL sweeper goroutine
// Redis implementation: SET key value EX ttl_seconds, GET key
```

**Cache Invalidation:** TTL expiry only (no active invalidation in MVP). Cache entries expire automatically. Admin endpoint `DELETE /admin/cache/{tool_name}` can flush entries for a specific tool.

**Redis Schema:**
```
Key: cache:{tool_name}:{args_hash}
Value: JSON-serialized tool result
TTL: CacheTTL seconds
```

---

**Responsibility:** Extract session ID from request, load context, attach to request.

**Extraction Logic:**
```
Priority (try in order):
1. Header: X-Session-ID
2. Cookie: session_id
3. Generate new if not found (caller must explicitly create)

If session ID found:
  session = session_store.GetSession(session_id)
  if session.ExpiresAt < now {
    return error "session expired"
  }
  ctx = request.Context().WithValue("session", session)
  request = request.WithContext(ctx)
else:
  ctx = request.Context().WithValue("session", nil)
  request = request.WithContext(ctx)
```

**In Handlers:**
```go
func handleToolCall(w http.ResponseWriter, r *http.Request) {
  session := r.Context().Value("session").(*Session)
  if session == nil {
    // Create new session or return error
  }
  // ... use session ...
}
```

---

### 10. Rate Limiter (`src/ratelimit/limiter.go`)

**Responsibility:** Enforce usage limits to protect backend services and ensure fair access.

**Token Bucket Algorithm:**
```
State per session:
  - tokens: current token count (max = capacity)
  - last_refill: timestamp of last token refill

On request:
  elapsed = now - last_refill
  tokens += elapsed * (rate_per_second)
  tokens = min(tokens, capacity)
  last_refill = now

  if tokens >= cost (default 1):
    tokens -= cost
    allow()
  else:
    reject with 429 Too Many Requests
```

**Configuration:**
```
Global:
  rate_limit_requests = 1000    // tokens per session
  rate_limit_window_seconds = 60  // refill period

Per-service (override global):
  weather_api:
    rate_limit_requests = 100
    rate_limit_window_seconds = 60
```

**Response:**
```
HTTP 429 Too Many Requests
Content-Type: application/json

{
  "error": "rate_limit_exceeded",
  "retry_after": 5,  // seconds
  "reset_at": "2026-05-13T12:34:56Z"
}

// Also set HTTP header:
Retry-After: 5
```

---

### 11. Logger (`src/logging/logger.go`)

**Responsibility:** Structured JSON logging with context for debugging and monitoring.

**Log Format:**
```json
{
  "timestamp": "2026-05-13T12:34:56.789Z",
  "level": "info",
  "message": "tool_call_started",
  "trace_id": "550e8400-e29b-41d4-a716-446655440000",
  "session_id": "session_abc123",
  "request_id": "req_xyz789",
  "service": "mcp-platform",
  "component": "router",
  "tool_name": "weather_getCurrentWeather",
  "instance": "mcp_server_1:9000",
  "latency_ms": 45,
  "error": null,
  "custom_field": "value"
}
```

**Log Levels:**
- **DEBUG** — Detailed diagnostic info (not in production)
- **INFO** — Normal operation events (service starts, tools registered)
- **WARN** — Recoverable issues (service went down, recovered)
- **ERROR** — Error conditions (service unhealthy, invalid request)

**Sampling (for high-volume logs):**
- Log every request at INFO level
- Sample DEBUG logs (1 per 100 requests)
- Always log errors

---

### 12. Tracer (`src/tracing/tracer.go`)

**Responsibility:** Distributed tracing using OpenTelemetry, end-to-end visibility.

**Span Hierarchy:**
```
Span: http_request (root)
  ├─ Span: session_lookup
  ├─ Span: rate_limit_check
  ├─ Span: tool_route_lookup
  ├─ Span: load_balance_select
  └─ Span: backend_call
      ├─ Span: adapter_convert_request
      ├─ Span: network_io
      └─ Span: adapter_convert_response
```

**Span Attributes:**
```
Root span (http_request):
  - http.method = "POST"
  - http.url = "http://router:8080/mcp/call"
  - http.status_code = 200
  - trace_id = "550e8400-e29b-41d4-a716-446655440000"
  - span_id = "f9d9f9d9f9d9f9d9"

Tool call span (backend_call):
  - tool.name = "weather_getCurrentWeather"
  - service.name = "weather_api_rest"
  - service.instance = "weather_api:8888"
  - latency_ms = 123
  - result_type = "success"
```

**Export:**
- OpenTelemetry Protocol (OTLP) over gRPC
- Endpoint: `$OTEL_EXPORTER_OTLP_ENDPOINT` (e.g., localhost:4317)
- Batching: Collect spans, export every 5 seconds or 512 spans

---

## Data Flow

### Scenario 1: Successful MCP Tool Call

```
1. Agent → Router
   POST /mcp/call
   {
     "tool_name": "weather_getCurrentWeather",
     "arguments": { "city": "San Francisco" }
   }
   Headers: X-Session-ID: session_123, Authorization: Bearer token_xyz

2. Router Middleware
   a) Create trace span (http_request)
   b) Log: "http_request_received"
   c) Extract session from X-Session-ID header
   d) Load session from Redis (if not found, error)
   e) Check rate limit: allow if tokens available

3. Router Core
   a) Registry.GetServiceByToolName("weather_getCurrentWeather")
      → Returns MCP server at localhost:9000
   b) LoadBalancer.SelectInstance()
      → Returns healthy instance
   c) Create span (backend_call)

4. MCP Adapter
   a) Connect to MCP server if not connected
   b) Build MCP message:
      {
        "jsonrpc": "2.0",
        "method": "tools/call",
        "params": {
          "name": "weather_getCurrentWeather",
          "arguments": { "city": "San Francisco" }
        },
        "id": "call_001"
      }
   c) Send over MCP protocol
   d) Wait for response (timeout 10s)
   e) Receive:
      {
        "jsonrpc": "2.0",
        "result": {
          "toolResult": {
            "type": "text",
            "text": "San Francisco: 72°F, partly cloudy"
          }
        },
        "id": "call_001"
      }

5. Router → Response
   a) Extract result from MCP response
   b) Update session: session.ToolInvocations.append(invocation)
   c) SessionStore.UpdateSession(session)
   d) End trace span
   e) Log: "tool_call_completed", latency=123ms, result_type=success
   f) Return HTTP 200 with response:
      {
        "result": "San Francisco: 72°F, partly cloudy",
        "session_id": "session_123",
        "trace_id": "550e8400-e29b-41d4-a716-446655440000"
      }

6. OpenTelemetry Exporter
   a) Batch spans and send to Jaeger/DataDog
```

---

### Scenario 2: Successful REST API Tool Call (via Adapter)

```
1. Agent → Router
   POST /mcp/call
   {
     "tool_name": "stock_getPrice",
     "arguments": { "symbol": "GOOGL" }
   }

2. Router Middleware
   (same as Scenario 1, steps 2a-2e)

3. Router Core
   a) Registry.GetServiceByToolName("stock_getPrice")
      → Returns REST service at api.stocks.com
   b) LoadBalancer.SelectInstance()
      → Returns healthy instance
   c) Route to REST adapter

4. REST Adapter
   a) REST metadata lookup: stock_getPrice
      {
        "http_method": "GET",
        "url_pattern": "/api/v1/stock/{symbol}/price",
        "query_params": ["symbol"],
        "auth": { "type": "API_KEY", "header": "X-API-Key" }
      }
   b) Build HTTP request:
      GET /api/v1/stock/GOOGL/price HTTP/1.1
      Host: api.stocks.com
      X-API-Key: secret_key_123
   c) Execute HTTP call (timeout 10s)
   d) Receive HTTP 200:
      {
        "symbol": "GOOGL",
        "price": 185.50,
        "currency": "USD"
      }
   e) Convert response to MCP format:
      {
        "type": "text",
        "text": "GOOGL: $185.50 USD"
      }

5. Router → Response
   (same as Scenario 1, steps 5a-5f)
```

---

### Scenario 3: Service Unavailable (All Instances Unhealthy)

```
1. Agent → Router
   POST /mcp/call with tool_name "unavailable_tool"

2. Router Core
   a) Registry.GetServiceByToolName("unavailable_tool")
      → Returns service "service_xyz"
   b) LoadBalancer.SelectInstance()
      → FilterHealthyInstances() returns empty list
      → Return error ErrNoHealthyInstances

3. Router → Response
   HTTP 503 Service Unavailable
   {
     "error": "service_unavailable",
     "message": "service_xyz has no healthy instances",
     "trace_id": "550e8400-e29b-41d4-a716-446655440000"
   }

4. Health Checker (background)
   a) Periodic health check to service_xyz instances
   b) Instances respond normally
   c) Mark instances as HEALTHY
   d) Log: "service_recovered"

5. Next Request
   → Succeeds as instances are now healthy
```

---

## Concurrency Model

### Goroutine Architecture

```
main():
  ├─ HTTP Server (blocking)
  │  └─ Request Handler (per request, goroutine)
  │
  ├─ Health Checker (periodic, goroutine)
  │  └─ Runs every 30 seconds
  │
  ├─ Trace Exporter (background, goroutine)
  │  └─ Batches spans, exports every 5s
  │
  └─ Signal Handler (blocking)
     └─ Graceful shutdown on SIGTERM/SIGINT
```

### Request Handler Flow (Single Goroutine)

```go
func handleToolCall(w http.ResponseWriter, r *http.Request) {
  // Synchronous execution
  session := loadSession(r)           // Wait for Redis
  checkRateLimit(session)              // Local operation
  service := registry.Lookup(toolName) // Local operation
  instance := loadBalancer.Select()    // Local operation
  result := adapter.Call(instance)     // Wait for network (10s timeout)
  session.Update(result)                // Wait for Redis
  w.Write(result)                       // Send response
}
```

### Concurrency Safety

| Component | Concurrency Pattern | Safety Mechanism |
|-----------|-------------------|------------------|
| Registry | Read-heavy | RWMutex (R lock for reads, W lock for registration) |
| Session Store | Read/write via Redis | Redis atomic operations |
| Load Balancer | Per-instance selection | Atomic counter for round-robin |
| Circuit Breaker | State transitions | sync.RWMutex per breaker |
| Response Cache | Read/write | sync.Map (in-memory) or Redis atomic ops |
| SSE Connections | Concurrent streams | sync.Map for connection registry |
| Rate Limiter | Per-session tokens | Session stored in Redis (atomic) |
| Trace Exporter | Batch accumulation | Buffered channel + mutex |
| Logger | Log output | Thread-safe writer (os.Stdout or file) |
| HTTP Server | Standard Go | Built-in goroutine-per-request |

---

## Error Handling

### Error Classification

**Client Errors (4xx):**
- 400 Bad Request — Invalid input, schema validation failed
- 404 Not Found — Tool name not registered
- 409 Conflict — Ambiguous bare tool name; use namespaced form `{service_id}::{tool_name}`
- 429 Too Many Requests — Rate limit exceeded
- 440 Session Not Found — Invalid session ID

**Server Errors (5xx):**
- 500 Internal Server Error — Unexpected condition
- 502 Bad Gateway — Backend error (MCP, REST, or gRPC)
- 503 Service Unavailable — No healthy instances **or circuit breaker is OPEN**
- 504 Gateway Timeout — Backend took too long

### Error Response Format

```json
{
  "error": "error_code",
  "message": "Human-readable error message",
  "trace_id": "550e8400-e29b-41d4-a716-446655440000",
  "details": {
    "service": "weather_api",
    "instance": "weather_1:9000",
    "backend_error": "connection refused"
  }
}
```

### Retry Logic

**Router retries (automatic):**
- Network timeout → Retry with next healthy instance (max 3 attempts)
- MCP parse error → Retry immediately (1 attempt)
- REST API 5xx → Retry with exponential backoff (max 2 attempts)

**Client retries (manual):**
- 429 Rate Limited → Wait Retry-After seconds, retry
- 503 Service Unavailable → Wait 30s, retry
- 504 Gateway Timeout → Wait 10s, retry

---

## Scalability Considerations

### Horizontal Scaling

**Stateless Router:**
- All routers can be replaced; no sticky sessions
- Easily scale up/down with Kubernetes or Docker Swarm

**Redis Session Store:**
- Redis cluster for HA (e.g., 3 primary + 3 replica)
- Sessions survive router restarts

**Load Balancing (External):**
- Use external LB (nginx, HAProxy, AWS ELB)
- Route incoming requests to multiple router instances

### Vertical Scaling

**Memory:**
- Registry (in-memory): ~1KB per service, ~100KB per 100 services
- Rate limiters: ~100B per active session
- Typical: <50MB at rest

**CPU:**
- Each goroutine (request) takes minimal CPU
- Go scheduler efficiently multiplexes goroutines on OS threads

**Network Throughput:**
- Typical request: 1-2KB
- Typical response: 5-10KB
- 1Gbps NIC supports ~10K requests/sec

### Bottlenecks & Mitigations

| Bottleneck | Symptom | Mitigation |
|-----------|---------|-----------|
| Redis latency | Session load slow | Redis cluster, pipeline operations |
| MCP server capacity | Backend overloaded | Add more instances, load balance |
| Network latency | Requests hang | Increase timeout, add retries |
| Trace export lag | Spans not appearing | Increase batch size, async export |
| Rate limiter token contention | Session collisions | Move rate limiter to Redis (atomic) |
| Backend failure cascade | All requests fail | Circuit breaker: fail-fast on OPEN, recover via HALF-OPEN |
| Cache miss storm | Thundering herd on cold start | Pre-warm cache at startup, stagger TTL expiry with jitter |
| Circuit breaker flapping | Rapid OPEN/CLOSE cycles | Tune failure threshold and cooldown window |
| In-memory cache growth | Memory exhaustion | Cap max entries, evict LRU, prefer Redis backend in prod |

---

**Next:** See [docs/PROTOCOL.md](PROTOCOL.md) for MCP protocol details.
