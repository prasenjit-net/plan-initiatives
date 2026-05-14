# MCP Platform Initiative

## Overview

The MCP Platform is a critical infrastructure component designed to consolidate, route, and adapt Model Context Protocol (MCP) resources and tools from multiple distributed sources—both MCP servers and traditional REST APIs—into a unified interface for AI agents.

### Problem Statement

Modern AI systems need access to diverse tools and resources scattered across multiple endpoints. These services may expose themselves as:
- **MCP Servers**: Native MCP protocol implementations
- **REST APIs**: Legacy or new HTTP-based services

Currently, there's no unified way to aggregate these heterogeneous resources and present them consistently to AI agents. The MCP Platform solves this by acting as a central hub that:
1. Discovers and registers services
2. Normalizes their capabilities into a common format
3. Routes tool invocations to the correct backend
4. Adapts protocol differences (MCP ↔ REST)
5. Manages cross-cutting concerns (sessions, rate limiting, observability)

---

## Initiative Goals

### Primary Objectives
1. **Unified Resource Aggregation** — Collect and present tools/resources from multiple MCP servers and REST APIs as a single consolidated interface
2. **Protocol Translation** — Enable seamless bidirectional conversion between MCP protocol and REST/HTTP
3. **Intelligent Routing** — Route tool invocations to appropriate backend service based on tool registry
4. **Session Management** — Maintain conversation context and user state across multiple requests
5. **Observability** — Provide complete visibility into router operations (traces, logs, metrics)
6. **Reliability** — Support load balancing, failover, and health checking for high availability
7. **Rate Limiting** — Protect backend services and enforce usage policies

### Success Criteria
- ✓ All registered services discoverable by agents via standardized MCP interface
- ✓ Tool invocation successfully routes to correct backend and returns result
- ✓ MCP server calls work end-to-end through the router
- ✓ REST API calls work end-to-end via MCP↔REST adapter
- ✓ Session context persists across multiple tool invocations
- ✓ Rate limiting prevents resource exhaustion
- ✓ Distributed tracing shows complete request flow
- ✓ Load balancer successfully distributes traffic across multiple backend instances
- ✓ Router can be deployed via Docker with full observability

---

## Architecture & Design

### High-Level Architecture

```
             ┌─────────────┐
             │  AI Agent   │
             └──────┬──────┘
       SSE Transport│  or  Stateless HTTP
             ┌──────▼──────────────────────────────────────────┐
             │             MCP PLATFORM                         │
             │  ┌──────────────────────────────────────────┐   │
             │  │  HTTP Server + SSE Transport Layer        │   │
             │  │  ├─ GET /mcp  (SSE stream, recommended)  │   │
             │  │  ├─ POST /mcp (SSE session requests)     │   │
             │  │  ├─ POST /mcp/call  (stateless HTTP)     │   │
             │  │  └─ GET /mcp/resources, GET /mcp/prompts │   │
             │  └──────────────────────────────────────────┘   │
             │                     ▲                            │
             │  ┌──────────────────┼───────────────────────┐   │
             │  │ Middleware Stack                          │   │
             │  ├──────────────────────────────────────────┤   │
             │  │ 1. Request Logging                       │   │
             │  │ 2. Tracing (Span Creation)               │   │
             │  │ 3. Session Extraction                    │   │
             │  │ 4. Rate Limiting                         │   │
             │  │ 5. Request Validation                    │   │
             │  └──────────────────────────────────────────┘   │
             │                     ▼                            │
             │  ┌──────────────────────────────────────────┐   │
             │  │  Core Router                             │   │
             │  │  ├─ Namespace Resolution                 │   │
             │  │  ├─ Response Cache (HIT → return early) │   │
             │  │  ├─ Service Registry Lookup              │   │
             │  │  ├─ Load Balancing                       │   │
             │  │  └─ Circuit Breaker (OPEN → fail-fast)  │   │
             │  └──────────────────────────────────────────┘   │
             │                     ▼                            │
             │  ┌──────────┬───────────────┬──────────────┐    │
             │  │          │               │              │    │
             │  ▼          ▼               ▼              │    │
             │ ┌────────┐ ┌────────┐ ┌──────────┐        │    │
             │ │  MCP   │ │  REST  │ │  gRPC    │        │    │
             │ │Adapter │ │Adapter │ │ Adapter  │        │    │
             │ └───┬────┘ └───┬────┘ └────┬─────┘        │    │
             └─────┼──────────┼───────────┼──────────────┘    │
                   │          │           │
                   ▼          ▼           ▼
           ┌────────────┐ ┌──────────┐ ┌───────────────┐
           │MCP Server1 │ │REST API  │ │gRPC Service A │
           │  (Port X)  │ │Endpoint  │ │  (Port 9090)  │
           └────────────┘ └──────────┘ └───────────────┘

External Services:
  ▪ Redis (Session Store + Response Cache)
  ▪ Tracing Backend (Jaeger/DataDog)
  ▪ Metrics (Prometheus/Datadog)
```
  ▪ Tracing Backend (Jaeger/DataDog)
  ▪ Log Aggregator (ELK/Datadog)
```

### Key Components

| Component | Purpose | Details |
|-----------|---------|---------|
| **HTTP Server** | Entry point for agents | Handles REST requests, serves as bridge to MCP protocol |
| **SSE Transport** | Persistent MCP connections | `GET /mcp` SSE stream + `POST /mcp` for requests; standard MCP transport per spec |
| **Service Registry** | Tracks available services | In-memory registry with namespaced tools from MCP servers, REST APIs, and gRPC services |
| **Router Core** | Routes requests | Namespace resolution, cache check, service lookup, circuit breaker, adapter dispatch |
| **MCP Protocol Handler** | MCP message parsing | Serializes/deserializes MCP messages, handles MCP handshake |
| **REST ↔ MCP Adapter** | Protocol translation | Converts REST API schemas to MCP tools, handles bidirectional conversion |
| **gRPC Adapter** | gRPC backend integration | Invokes gRPC services as MCP tools via Server Reflection or protobuf descriptors |
| **Load Balancer** | Distributes traffic | Routes to healthy instances of same service |
| **Circuit Breaker** | Failure isolation | CLOSED/OPEN/HALF-OPEN states; prevents cascading failures from slow/failing backends |
| **Health Checker** | Monitors service health | Periodic pings, marks services as healthy/unhealthy |
| **Session Store** | Manages state | Redis-backed persistent session storage |
| **Response Cache** | Tool result caching | In-memory or Redis-backed TTL cache for read-only tools; opt-in per tool |
| **Session Middleware** | Context injection | Extracts session from request, provides to handlers |
| **Rate Limiter** | Protects resources | Token bucket limiter, enforces per-session/per-IP policies |
| **Logger** | Structured logging | JSON logs with request ID, session ID, service context |
| **Tracer** | Distributed tracing | OpenTelemetry spans for complete request flow visibility |
| **Error Handler** | Standardized errors | Maps internal errors to HTTP status codes, includes correlation IDs |

---

## Implementation Plan

### Phase 1: Project Foundation & Infrastructure Setup
**Goal:** Establish project structure, configuration framework, and local development environment.

**Deliverables:**
- Project directory layout (src, tests, docs, scripts, docker)
- `go.mod` with pinned dependencies
- Configuration management (.env, environment variables)
- docker-compose.yml for local development
- GitHub CI/CD workflow template (.github/workflows)

**Rationale:** Before writing any code, establish the scaffolding and build pipeline to ensure consistent, reproducible development.

---

### Phase 2: MCP Protocol Implementation & Service Registry
**Goal:** Build the foundation for MCP protocol handling, service discovery, and the SSE transport.

**Deliverables:**
- **MCP Protocol Handler** (`src/mcp/protocol.go`): 
  - Parse/serialize MCP messages (tools, resources, prompts, calls)
  - Implement MCP initialization handshake
  - Validate message formats
  
- **SSE Transport Handler** (`src/transport/sse.go`):
  - `GET /mcp` — persistent SSE connection endpoint
  - `POST /mcp` — accept client requests bound to an SSE session
  - Heartbeat goroutine (ping every 30s)
  - Connection registry with configurable max connections
  
- **Service Registry** (`src/registry/registry.go`):
  - Register MCP servers, REST API endpoints, and gRPC services
  - Enforce `{service_id}::{tool_name}` namespacing on all tools
  - Query by bare name (with ambiguity detection) or namespaced name
  - Track service health status
  - Support dynamic registration/deregistration

**Rationale:** The registry is the central source of truth. SSE transport must be designed early since it affects how agents interact with the platform.

---

### Phase 3: Core Routing & Basic MCP Server Integration
**Goal:** Build the core routing engine and direct MCP server integration.

**Deliverables:**
- **Router Core** (`src/router/router.go`):
  - Accept tool invocations and route to correct backend
  - Handle resource/prompt queries
  - Manage request/response orchestration
  
- **MCP Server Client** (`src/adapters/mcp/client.go`):
  - Establish connections to remote MCP servers
  - Send MCP protocol messages to servers
  - Handle reconnection logic with exponential backoff

**Rationale:** Verify basic routing works before adding complexity (adapters, load balancing).

---

### Phase 4: Adapters & Protocol Translation
**Goal:** Enable the router to work with REST APIs and gRPC services as MCP tools.

**Deliverables:**
- **REST Client Adapter** (`src/adapters/rest/client.go`):
  - Parse OpenAPI/JSON Schema specs from REST APIs
  - Convert REST API endpoints to MCP-compatible namespaced tool definitions
  - Execute REST calls (GET, POST, etc.)
  - Handle various auth methods (API key, OAuth, mTLS)
  
- **REST ↔ MCP Converter** (`src/adapters/rest/converter.go`):
  - MCP tool call → HTTP request (URL, headers, body)
  - HTTP response → MCP tool result
  - Error mapping between protocols
  
- **gRPC Adapter** (`src/adapters/grpc/client.go`):
  - Discover methods via gRPC Server Reflection API (or protobuf descriptor fallback)
  - Map gRPC methods to namespaced MCP tool definitions
  - Execute gRPC invocations via persistent channels
  - Map gRPC status codes to MCP/HTTP errors
  - Support TLS, mTLS, and token metadata auth

**Rationale:** REST and gRPC adapters dramatically expand router capabilities, enabling integration with legacy HTTP services and modern microservices.

---

### Phase 5: Load Balancing, Health Checking & Circuit Breaker
**Goal:** Support multiple instances of same service for scalability and resilience.

**Deliverables:**
- **Load Balancer** (`src/loadbalancer/balancer.go`):
  - Maintain list of healthy instances per service
  - Distribute requests using round-robin or least-connections
  - Exclude unhealthy instances
  
- **Health Checker** (`src/health/checker.go`):
  - Periodic health checks to all services (configurable interval)
  - Mark services as healthy/unhealthy based on response
  - Integrate health status into registry
  
- **Circuit Breaker** (`src/circuitbreaker/breaker.go`):
  - CLOSED/OPEN/HALF-OPEN state machine per service instance
  - CLOSED → OPEN: 5 consecutive failures or >50% error rate in 10s window
  - OPEN: Fail-fast with HTTP 503 and `Retry-After` header (no backend call)
  - HALF-OPEN → CLOSED/OPEN: based on single probe request result
  - Configurable via `CIRCUIT_BREAKER_FAILURE_THRESHOLD` and `CIRCUIT_BREAKER_TIMEOUT_SECONDS`

**Rationale:** Enables high availability and protects the platform from cascading failures when backends degrade.

---

### Phase 6: Session Management & Response Cache (Redis-Backed)
**Goal:** Maintain user/conversation context across requests and reduce backend load with caching.

**Deliverables:**
- **Session Store** (`src/session/store.go`):
  - Connect to Redis with configurable connection pool
  - Create new sessions with unique IDs
  - Store session data (user context, conversation history, etc.)
  - Retrieve and update sessions
  - Handle session expiration (TTL)
  - Delete sessions on logout
  
- **Session Middleware** (`src/middleware/session.go`):
  - Extract session ID from request (header or cookie)
  - Load session context from Redis
  - Attach context to request
  
- **Response Cache** (`src/cache/cache.go`):
  - In-memory backend (single-instance) and Redis backend (multi-instance)
  - Cache key: `cache:{namespaced_tool}:{sha256(arguments)}`
  - Per-tool opt-in TTL via `ToolDefinition.CacheTTL`
  - Return `X-Cache: HIT/MISS` header and cache metadata in response
  - Admin flush endpoint: `DELETE /admin/cache/{tool_name}`

**Rationale:** Redis provides distributed session state. Response caching reduces backend calls for read-only tools and improves P50 latency significantly.

---

### Phase 7: Observability - Logging, Tracing, Rate Limiting
**Goal:** Provide complete visibility into router operations and protect resources.

**Deliverables:**
- **Structured Logger** (`src/logging/logger.go`):
  - JSON-formatted logs at DEBUG/INFO/WARN/ERROR levels
  - Include request ID, session ID, service name, latency in every log
  - Integration hooks for log aggregation (ELK, Datadog, etc.)
  - Sampling for high-volume requests
  
- **Distributed Tracer** (`src/tracing/tracer.go`):
  - OpenTelemetry SDK integration
  - Create spans for each operation (registry lookup, adapter conversion, backend call)
  - Propagate trace context (correlation IDs) across services
  - Measure and record latency metrics
  - Export traces to backend (Jaeger, Datadog, etc.)
  
- **Rate Limiter** (`src/ratelimit/limiter.go`):
  - Token bucket algorithm implementation
  - Per-session rate limiting
  - Per-IP rate limiting (optional)
  - Per-service rate limiting
  - Return 429 Too Many Requests with Retry-After header
  - Configurable policies per service

**Rationale:** Production systems need visibility and protection. These components ensure the router is observable and resilient.

---

### Phase 8: HTTP Server & Integration
**Goal:** Wire up all components into a functioning HTTP server.

**Deliverables:**
- **HTTP Server** (`src/server/server.go`):
  - Start HTTP server on configurable port (default 8080)
  - TLS support (optional, for production)
  - Graceful shutdown handling
  
- **HTTP Routes & Handlers** (`src/server/handlers.go`):
  - `POST /mcp/call` — Invoke a tool (MCP tool call endpoint)
  - `GET /mcp/resources` — List available resources/tools
  - `GET /mcp/prompts` — List available prompts
  - `POST /sessions` — Create new session
  - `GET /sessions/{id}` — Retrieve session
  - `DELETE /sessions/{id}` — Delete session
  - `GET /health` — Health check endpoint (for load balancers)
  - `GET /ready` — Readiness check (dependencies ready)
  
- **Middleware Chain** (`src/middleware/`):
  - Middleware stack: Logging → Tracing → Session → Rate Limiting → Auth (if needed) → Handler
  - Request validation and sanitization
  - Error handling and response formatting
  
- **Error Handler** (`src/server/errors.go`):
  - Standardized error response format
  - HTTP status code mapping
  - Include correlation ID in all error responses

**Rationale:** This phase brings all components together into a working HTTP service.

---

### Phase 9: Testing & Documentation
**Goal:** Ensure correctness and provide operational guidance.

**Deliverables:**
- **Unit Tests** (`tests/unit/`):
  - Protocol handler: MCP message parsing, validation
  - Service registry: Registration, lookup, health status
  - Router: Tool routing logic, error cases
  - Adapters: REST schema parsing, MCP↔REST conversion
  - Load balancer: Round-robin distribution, instance selection
  - Session store: Create, retrieve, update, expire sessions
  - Rate limiter: Token consumption, 429 responses
  - Middleware: Context propagation, error handling
  - Target: 80%+ code coverage on core logic
  
- **Integration Tests** (`tests/integration/`):
  - Full end-to-end: Agent → Router → MCP Server → Response
  - Full end-to-end: Agent → Router → REST API (via adapter) → Response
  - Session persistence: Multi-request conversation flow
  - Load balancer failover: Traffic reroutes to healthy instance
  - Rate limiting: Requests rejected after limit exceeded
  - Redis interaction: Session store correctly uses Redis
  - Tracing: Spans exported to tracing backend
  
- **Documentation**:
  - **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** — Component architecture, data flow diagrams, design patterns
  - **[docs/PROTOCOL.md](docs/PROTOCOL.md)** — MCP protocol details, message formats, handshake sequence
  - **[docs/API.md](docs/API.md)** — HTTP API endpoints, request/response examples, error codes
  - **[docs/DEPLOYMENT.md](docs/DEPLOYMENT.md)** — Docker setup, environment configuration, scaling guidelines
  - **[docs/DEVELOPMENT.md](docs/DEVELOPMENT.md)** — Local setup, testing, debugging

---

### Phase 10: Deployment & DevOps
**Goal:** Enable production deployment and monitoring.

**Deliverables:**
- **Docker** (`docker/Dockerfile`):
  - Multi-stage build (smaller image)
  - Non-root user for security
  - Health check command
  
- **Docker Compose** (`docker-compose.yml`):
  - Router service
  - Redis service
  - Optional: Test MCP server
  - Optional: Jaeger/tracing backend
  - Volume mounts for config and logs
  
- **Configuration Management**:
  - `.env.example` — Template with all configurable parameters
  - `config/router.yaml` — Example configuration file
  - Validation on startup (fail-fast if missing required config)
  
- **Scripts** (`scripts/`):
  - `setup.sh` — One-command local environment setup
  - `build.sh` — Build Docker image
  - `deploy.sh` — Deployment helper (k8s manifest templates)
  - `health-check.sh` — Manual health verification

---

## Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Language** | Go 1.21+ | Performance, concurrency (goroutines), static typing, fast compilation |
| **Web Framework** | Gin or gorilla/mux | Lightweight HTTP routing |
| **MCP SDK** | Official Anthropic MCP SDK for Go | Standard MCP protocol implementation |
| **SSE Transport** | Go standard `http.Flusher` | Built-in SSE support; no extra dependency |
| **REST Client** | go-resty or http.Client | Simple, reliable HTTP communication |
| **gRPC Client** | google.golang.org/grpc | gRPC client for backend services |
| **Circuit Breaker** | sony/gobreaker | Industry-standard circuit breaker for Go |
| **Session Store** | Redis | Distributed session state, supports multi-instance router |
| **Response Cache** | In-memory (sync.Map) + Redis | TTL-based tool result caching; Redis for multi-instance |
| **Tracing** | OpenTelemetry | Vendor-agnostic, industry standard |
| **Logging** | slog or zap | Structured logging, performance |
| **Rate Limiting** | Token bucket (custom) | Simple, configurable, predictable |
| **Testing** | Go testing + testify | Built-in testing, assertion helpers |
| **Docker** | Docker CE | Container orchestration, reproducibility |
| **CI/CD** | GitHub Actions | Integrated with repository, free for public repos |

---

## Configuration & Environment

### Required Configuration
```
MCP_ROUTER_PORT=8080
MCP_ROUTER_ENV=development|production
MCP_TRANSPORT=sse|http          # default: sse

# Redis Session Store + Cache
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0
REDIS_PASSWORD=

# MCP Servers (comma-separated or JSON)
MCP_SERVERS=server1:localhost:9000,server2:localhost:9001

# REST API Endpoints
REST_ENDPOINTS=weather_api:https://api.weather.com,stock_api:https://api.stocks.com

# gRPC Services
GRPC_SERVERS=payment_svc:localhost:9090,inventory_svc:localhost:9091

# SSE Transport
MCP_SSE_MAX_CONNECTIONS=1000

# Tracing
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
OTEL_SERVICE_NAME=mcp-platform

# Logging
LOG_LEVEL=info
LOG_FORMAT=json

# Rate Limiting
RATE_LIMIT_REQUESTS=1000
RATE_LIMIT_WINDOW_SECONDS=60

# Circuit Breaker
CIRCUIT_BREAKER_FAILURE_THRESHOLD=5
CIRCUIT_BREAKER_TIMEOUT_SECONDS=30

# Response Cache
CACHE_BACKEND=memory|redis      # default: memory
CACHE_DEFAULT_TTL_SECONDS=0     # 0 = disabled; per-tool TTL takes precedence
```

### Deployment Scenarios
1. **Local Development** — docker-compose with Redis, optional mock MCP server
2. **Single Instance** — Docker container with Redis backend
3. **Clustered** — Multiple router instances + Redis cluster + load balancer
4. **Kubernetes** — Helm charts, StatefulSet for Redis, Deployment for router

---

## Dependencies & Assumptions

### Technical Assumptions
1. **MCP Servers** — Assume standard MCP protocol implementation (can discover capabilities via introspection)
2. **REST APIs** — Assume OpenAPI 3.0 or JSON Schema for capability discovery; fallback to manual configuration
3. **Redis** — Assume Redis 6.0+ with basic key-value operations
4. **Network** — Assume reliable network between router and backend services; implement retry logic for transient failures
5. **Concurrency** — Assume Go runtime can handle thousands of concurrent requests (goroutines)

### External Dependencies
- MCP Protocol Specification (from Anthropic)
- OpenAPI/JSON Schema specifications (from services being integrated)
- Redis documentation
- OpenTelemetry documentation

---

## Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|-----------|
| MCP Server goes offline | Requests fail | Health checking, failover to replica, graceful error messages |
| Backend cascading failure | All tool calls fail | Circuit breaker: fail-fast when OPEN, probe via HALF-OPEN |
| Redis unavailable | Sessions lost | Redis persistence, backup instance, in-memory fallback (temporary) |
| Latency from adapter conversion | Requests slow | Response cache for read-only tools, schema caching |
| Cache staleness | Stale tool results | Short TTLs (per-tool), opt-in caching, no caching for write operations |
| Rate limiter too strict | Legitimate traffic blocked | Configurable limits, logging of 429 responses for tuning |
| Trace overhead | Performance degradation | Sampling (10% of requests), async export to tracing backend |
| MCP protocol version mismatch | Integration fails | Protocol version negotiation, compatibility layer |
| gRPC reflection unavailable | gRPC tools not discoverable | Fallback to manually configured protobuf descriptor files |

---

## Success Metrics

### Functional Metrics
- ✓ 100% of registered services accessible via router
- ✓ Tool invocations succeed with >99% success rate
- ✓ Session persistence: 100% of sessions survive across requests

### Performance Metrics
- ✓ Latency: P50 < 50ms, P99 < 200ms (end-to-end)
- ✓ Throughput: Support >1000 requests/sec
- ✓ Memory: <200MB at rest, <500MB under load

### Reliability Metrics
- ✓ Uptime: >99.5% availability
- ✓ Error rate: <0.1% (excluding user errors)
- ✓ Failover time: <5 seconds when backend fails

### Observability Metrics
- ✓ Tracing: 100% of requests traced end-to-end
- ✓ Logging: All significant events logged with context
- ✓ Rate limiting: Accurate token consumption tracking

---

## Open Design Questions (For Future Refinement)

1. **Service Discovery**
   - Should the router use static configuration or dynamic discovery (Consul, Kubernetes API)?
   - Decision: Start with static `.env` config; design for pluggable discovery later.

2. **Authentication to Backend Services**
   - How should the router authenticate to upstream MCP servers and REST APIs?
   - Options: API keys, mutual TLS, OAuth tokens
   - Decision: Support configurable per-service auth; start with API keys.

3. **Adapter Extensibility**
   - Should custom adapters be loadable as plugins, or hardcoded?
   - Decision: Hardcode REST + MCP adapters for MVP; use interface pattern for future plugins.

4. **Multi-Tenancy**
   - Should different agents/clients be isolated from each other?
   - Decision: Out of scope for MVP; use API keys or request headers for client identification.

5. **Caching**
   - Should frequently-used tool definitions be cached? Tool results?
   - Decision: Cache tool schemas (low TTL), not results; configurable cache backend.

---

## Timeline & Milestones

| Milestone | Phase(s) | Est. Effort | Dependencies |
|-----------|----------|-------------|--------------|
| **M1: Foundation Ready** | 1 | 1-2 weeks | None |
| **M2: MCP Integration** | 2-3 | 2-3 weeks | M1 |
| **M3: REST Adapter** | 4 | 2-3 weeks | M2 |
| **M4: Scalability** | 5-6 | 2-3 weeks | M2 |
| **M5: Observability** | 7 | 1-2 weeks | M4 |
| **M6: HTTP Server** | 8 | 1 week | M5 |
| **M7: Testing & Docs** | 9 | 2-3 weeks | M6 |
| **M8: Production Ready** | 10 | 1-2 weeks | M7 |
| **Total** | 1-10 | **14-20 weeks** | Sequential |

---

## File Structure (Final)

```
initiatives/mcp-platform/
├── README.md                          # This file
├── .env.example                       # Configuration template
├── go.mod                             # Go module definition
├── go.sum                             # Go dependency lock
├── Dockerfile                         # Container image
├── docker-compose.yml                 # Local dev environment
├── Makefile                           # Build commands
│
├── src/
│   ├── main.go                        # Entry point
│   ├── config/
│   │   └── config.go                  # Configuration loading
│   ├── mcp/
│   │   └── protocol.go                # MCP message handling
│   ├── registry/
│   │   └── registry.go                # Service registry
│   ├── router/
│   │   └── router.go                  # Core routing logic
│   ├── adapters/
│   │   ├── mcp/
│   │   │   └── client.go              # MCP server client
│   │   ├── rest/
│   │   │   ├── client.go              # REST API client
│   │   │   └── converter.go           # Protocol conversion
│   │   └── grpc/
│   │       └── client.go              # gRPC adapter
│   ├── transport/
│   │   └── sse.go                     # SSE transport handler
│   ├── circuitbreaker/
│   │   └── breaker.go                 # Circuit breaker (gobreaker)
│   ├── cache/
│   │   └── cache.go                   # Response cache (memory + Redis)
│   ├── loadbalancer/
│   │   └── balancer.go                # Load balancer
│   ├── health/
│   │   └── checker.go                 # Health checks
│   ├── session/
│   │   └── store.go                   # Redis session store
│   ├── middleware/
│   │   ├── session.go                 # Session extraction
│   │   ├── ratelimit.go               # Rate limiting
│   │   └── tracing.go                 # Tracing middleware
│   ├── logging/
│   │   └── logger.go                  # Structured logging
│   ├── tracing/
│   │   └── tracer.go                  # OpenTelemetry tracing
│   ├── ratelimit/
│   │   └── limiter.go                 # Token bucket limiter
│   ├── server/
│   │   ├── server.go                  # HTTP server setup
│   │   ├── handlers.go                # HTTP handlers
│   │   └── errors.go                  # Error handling
│   └── types/
│       └── types.go                   # Shared data types
│
├── tests/
│   ├── unit/
│   │   ├── mcp/
│   │   ├── registry/
│   │   ├── router/
│   │   ├── adapters/
│   │   ├── loadbalancer/
│   │   ├── session/
│   │   ├── ratelimit/
│   │   └── middleware/
│   └── integration/
│       ├── e2e_mcp_test.go            # End-to-end MCP test
│       ├── e2e_rest_test.go           # End-to-end REST test
│       ├── session_test.go            # Session persistence test
│       ├── loadbalancer_test.go       # Failover test
│       └── tracing_test.go            # Tracing export test
│
├── docs/
│   ├── ARCHITECTURE.md                # Component architecture
│   ├── PROTOCOL.md                    # MCP protocol details
│   ├── API.md                         # HTTP API reference
│   ├── DEPLOYMENT.md                  # Deployment guide
│   ├── DEVELOPMENT.md                 # Local development
│   └── DIAGRAMS.md                    # ASCII diagrams & flows
│
├── scripts/
│   ├── setup.sh                       # Local environment setup
│   ├── build.sh                       # Build Docker image
│   ├── deploy.sh                      # Deployment helper
│   └── health-check.sh                # Health verification
│
├── config/
│   ├── router.yaml                    # Example config file
│   └── redis-compose.yaml             # Redis-only compose
│
└── .github/
    └── workflows/
        ├── ci.yml                     # CI pipeline
        └── release.yml                # Release pipeline
```

---

## Next Steps

1. **Review & Approval** — Confirm this plan aligns with vision and constraints
2. **Dependency Resolution** — Clarify any unknowns (external API specs, existing standards)
3. **Implementation Kickoff** — Begin Phase 1 (project foundation)
4. **Incremental Delivery** — Phase by phase, with testing and documentation at each step

---

**Document Version:** 1.0  
**Date:** May 13, 2026  
**Status:** Planning (No Implementation Yet)
