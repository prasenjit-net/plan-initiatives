# Development Guide

## Quick Start

### Prerequisites
- Go 1.21+
- Docker & Docker Compose
- Redis (or use docker-compose)
- Git

### Setup (5 minutes)

```bash
cd initiatives/mcp-platform

# 1. Copy environment template
cp .env.example .env

# 2. Start dependencies
docker-compose up -d redis

# 3. Run router
go run ./src/main.go

# 4. In another terminal, test
curl http://localhost:8080/health
```

---

## Project Structure

```
.
├── src/
│   ├── main.go                    # Entry point
│   ├── config/
│   ├── mcp/                       # MCP protocol
│   ├── registry/                  # Service registry
│   ├── router/                    # Core routing
│   ├── adapters/                  # MCP & REST adapters
│   ├── loadbalancer/              # Load balancing
│   ├── session/                   # Session management
│   ├── middleware/                # HTTP middleware
│   ├── logging/                   # Logging
│   ├── tracing/                   # Tracing
│   ├── ratelimit/                 # Rate limiting
│   └── server/                    # HTTP server
├── tests/
│   ├── unit/                      # Unit tests
│   └── integration/               # Integration tests
├── docs/                          # Documentation
├── scripts/                       # Helper scripts
├── docker-compose.yml             # Development environment
├── Dockerfile                     # Container image
├── go.mod & go.sum               # Dependencies
└── README.md                      # Overview
```

---

## Development Workflow

### 1. Making Code Changes

**Example: Add a new MCP protocol method handler**

```bash
# 1. Edit the code
vim src/mcp/protocol.go

# 2. Run unit tests
go test ./src/mcp -v

# 3. Run integration tests (requires docker-compose up)
go test ./tests/integration -v

# 4. Build locally
go build -o bin/mcp-platform ./src/main.go

# 5. Test
./bin/mcp-platform
```

### 2. Running Tests

**Unit Tests** (fast, no external dependencies):
```bash
go test ./src/... -v -race -cover
```

**Integration Tests** (slow, requires docker-compose):
```bash
docker-compose up -d
go test ./tests/integration/... -v
docker-compose down
```

**Coverage Report:**
```bash
go test ./src/... -coverprofile=coverage.out
go tool cover -html=coverage.out  # Opens browser
```

### 3. Debugging

**Print Debugging:**
```go
import "log"

log.Printf("DEBUG: value=%v\n", value)
```

**Using Delve Debugger:**
```bash
go install github.com/go-delve/delve/cmd/dlv@latest

# Start debugger
dlv debug ./src/main.go

# In dlv prompt:
# > b main.main        # Set breakpoint at main()
# > c                  # Continue
# > n                  # Next line
# > p variable_name    # Print variable
# > quit               # Exit
```

**Logging:**
```bash
# View structured logs
go run ./src/main.go | jq .

# Filter by level
go run ./src/main.go | jq 'select(.level=="error")'

# Filter by message
go run ./src/main.go | jq 'select(.message | contains("tool_call"))'
```

### 4. Code Style & Linting

**Format Code:**
```bash
go fmt ./src/...
goimports -w ./src/  # Auto-add/remove imports
```

**Lint:**
```bash
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
golangci-lint run ./src/...
```

**Vet:**
```bash
go vet ./src/...
```

---

## Common Tasks

### Add a New Tool Source (MCP Server)

1. **Register in environment:**
   ```
   MCP_SERVERS=existing_server:host:9000,new_server:host:9001
   ```

2. **Implement MCP client integration** (if needed):
   ```go
   // src/adapters/mcp/client.go
   type MCPClient struct {
     // fields
   }

   func (c *MCPClient) ListTools() ([]ToolDefinition, error) {
     // Call MCP server
   }
   ```

3. **Test:**
   ```bash
   go run ./src/main.go
   curl http://localhost:8080/mcp/resources | jq .
   ```

### Add a New REST API Integration

1. **Add REST endpoint config:**
   ```
   REST_ENDPOINTS=existing_api:https://api1.com,new_api:https://api2.com
   ```

2. **Implement REST schema parser** (if needed):
   ```go
   // src/adapters/rest/client.go
   type RESTClient struct {
     // fields
   }

   func (c *RESTClient) ImportSchema(spec OpenAPISpec) ([]ToolDefinition, error) {
     // Parse OpenAPI spec
   }
   ```

3. **Test:**
   ```bash
   go run ./src/main.go
   curl http://localhost:8080/mcp/resources | jq '.tools[] | select(.name | contains("new_api"))'
   ```

### Add Middleware

1. **Create middleware file:**
   ```go
   // src/middleware/custom.go
   func CustomMiddleware(next http.Handler) http.Handler {
     return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
       // Do something before
       next.ServeHTTP(w, r)
       // Do something after
     })
   }
   ```

2. **Register in server:**
   ```go
   // src/server/server.go
   router.Use(CustomMiddleware)
   ```

3. **Test:**
   ```bash
   go test ./src/middleware/... -v
   ```

### Fix a Bug

1. **Write a failing test first:**
   ```go
   // tests/unit/router/router_test.go
   func TestRouterBugFix(t *testing.T) {
     // Reproduce bug
     result, err := router.RouteToolCall(...)
     assert.Error(t, err)  // Should fail initially
   }
   ```

2. **Make test pass:**
   ```go
   // src/router/router.go
   func (r *Router) RouteToolCall(...) {
     // Fix bug
   }
   ```

3. **Verify test passes:**
   ```bash
   go test ./tests/unit/router -v
   ```

---

## Performance Profiling

### CPU Profiling

```bash
# Run with CPU profile
go run -cpuprofile=cpu.prof ./src/main.go
# Let it run for a bit, then Ctrl+C

# Analyze
go tool pprof cpu.prof
# In pprof prompt:
# > top10          # Show top 10 functions by CPU
# > list function  # Show source of function
# > web            # Open in browser
```

### Memory Profiling

```bash
# Run with memory profile
go run -memprofile=mem.prof ./src/main.go
# Let it run for a bit, then Ctrl+C

# Analyze
go tool pprof mem.prof
# In pprof prompt:
# > top10          # Show top 10 memory allocators
# > alloc_space    # Total allocations
```

### Benchmarking

```bash
# Create benchmark test
// tests/unit/router/router_bench_test.go
func BenchmarkRouteToolCall(b *testing.B) {
  for i := 0; i < b.N; i++ {
    router.RouteToolCall(...)
  }
}

# Run
go test -bench=. ./tests/unit/router -benchmem
```

---

## Docker Development

### Build Docker Image

```bash
# Local build
docker build -f Dockerfile -t mcp-platform:dev .

# Run
docker run -p 8080:8080 --env-file .env mcp-platform:dev
```

### Docker Compose Development

```bash
# Start all services with hot-reload (if supported)
docker-compose up

# View logs
docker-compose logs -f mcp-platform

# Execute command in container
docker-compose exec mcp-platform go test ./... -v

# Stop
docker-compose down
```

---

## CI/CD Pipeline

### GitHub Actions

**.github/workflows/ci.yml:**
```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
    - uses: actions/checkout@v2
    - uses: actions/setup-go@v2
      with:
        go-version: 1.21

    - name: Run Linter
      run: |
        go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
        golangci-lint run ./src/...

    - name: Run Tests
      run: go test ./... -v -race -cover

    - name: Build
      run: go build -o bin/mcp-platform ./src/main.go

    - name: Build Docker Image
      run: docker build -t mcp-platform:latest .
```

---

## Debugging MCP Communication

### Capture MCP Messages

```go
// In mcp/protocol.go
func (p *Protocol) Debug(enable bool) {
  p.debug = enable
}

// Then log messages
if p.debug {
  log.Printf("MCP Send: %s\n", message)
  log.Printf("MCP Recv: %s\n", response)
}
```

### Use tcpdump to Capture Network Traffic

```bash
# On Linux
tcpdump -i lo -A 'tcp port 9000' | head -100

# On macOS
sudo tcpdump -i lo0 -A 'tcp port 9000' | head -100
```

---

## Git Workflow

### Branch Convention

```
main          # Stable, production-ready
├─ feature/*  # New features
├─ bugfix/*   # Bug fixes
├─ refactor/* # Code refactoring
└─ docs/*     # Documentation
```

### Commit Message Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:** feat, fix, refactor, test, docs, chore
**Example:**
```
feat(router): add tool routing logic

Implement service registry lookup and
load balancer integration for routing
tool calls to correct backend.

Closes #42
```

### Create a Feature Branch

```bash
git checkout -b feature/tool-routing
# ... make changes ...
git add src/router/router.go
git commit -m "feat(router): implement tool routing"
git push origin feature/tool-routing
# Create PR in GitHub
```

---

## Troubleshooting Development Issues

### Issue: "go: cannot find module"

```bash
go mod tidy  # Download missing dependencies
go mod vendor  # Vendorize dependencies
```

### Issue: "package not found"

```bash
# Add missing import
go get github.com/some/package

# Or import and run
go run ./src/main.go  # Auto-downloads
```

### Issue: Tests failing locally but passing in CI

```bash
# Run with same flags as CI
go test ./... -v -race -covermode=atomic

# Check for flaky tests
go test ./... -count=10  # Run 10 times
```

### Issue: Docker image too large

```bash
# Use multi-stage build
FROM golang:1.21 as builder
RUN go build ...

FROM alpine:latest
COPY --from=builder /app/mcp-platform /app/
```

---

## Release Process

```bash
# 1. Update version in go.mod
vim go.mod  # Update version field

# 2. Update CHANGELOG.md
vim CHANGELOG.md

# 3. Create release branch
git checkout -b release/v1.0.0

# 4. Tag release
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0

# 5. CI automatically builds and publishes docker image
```

---

## Resources

- [Go Documentation](https://golang.org/doc)
- [MCP Specification](https://modelcontextprotocol.io)
- [Docker Docs](https://docs.docker.com)
- [Kubernetes Docs](https://kubernetes.io/docs)
- [OpenTelemetry](https://opentelemetry.io)
