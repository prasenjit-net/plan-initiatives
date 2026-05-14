# MCP Platform Deployment Guide

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Local Development Setup](#local-development-setup)
3. [Docker Deployment](#docker-deployment)
4. [Kubernetes Deployment](#kubernetes-deployment)
5. [Configuration](#configuration)
6. [Monitoring & Observability](#monitoring--observability)
7. [Scaling & Performance](#scaling--performance)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### System Requirements
- **OS:** Linux, macOS, or Windows (with WSL2)
- **CPU:** 2+ cores recommended
- **Memory:** 4GB minimum, 8GB+ for production
- **Disk:** 10GB for docker images and logs

### Software Requirements
- **Go:** 1.21+
- **Docker:** 20.10+
- **Docker Compose:** 2.0+
- **Redis:** 6.0+ (or use docker-compose)
- **Git:** For version control

### Optional Tools
- **kubectl:** 1.20+ (for Kubernetes)
- **Helm:** 3.0+ (for Helm charts)
- **curl/HTTPie:** For API testing
- **jq:** For JSON processing

---

## Local Development Setup

### 1. Clone and Initialize

```bash
cd /path/to/plan-initiative/initiatives/mcp-platform
cp .env.example .env
```

### 2. Environment Configuration

Edit `.env`:
```
# Server
MCP_ROUTER_PORT=8080
MCP_ROUTER_ENV=development
MCP_ROUTER_LOG_LEVEL=debug

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0

# MCP Servers (comma-separated)
MCP_SERVERS=mcp_server_1:localhost:9000,mcp_server_2:localhost:9001

# REST APIs
REST_ENDPOINTS=weather_api:https://api.weather.example.com,stock_api:https://api.stocks.example.com

# Tracing
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
OTEL_SERVICE_NAME=mcp-platform
OTEL_EXPORTER_OTLP_HEADERS=

# Logging
LOG_LEVEL=debug
LOG_FORMAT=json
```

### 3. Start Dependencies (Redis)

```bash
docker-compose up -d redis
```

Verify:
```bash
docker-compose ps
redis-cli ping
# Output: PONG
```

### 4. Run Router in Development Mode

```bash
# Terminal 1: Start the router
go run ./src/main.go

# Output:
# 2026/05/13 12:34:56 [INFO] MCP Platform starting on :8080
# 2026/05/13 12:34:56 [INFO] Connected to Redis at localhost:6379
# 2026/05/13 12:34:56 [INFO] Registered 2 MCP servers
# 2026/05/13 12:34:56 [INFO] Registered 2 REST endpoints
# 2026/05/13 12:34:56 [INFO] Router ready
```

### 5. Test the Router

```bash
# Terminal 2: Test health
curl http://localhost:8080/health

# Response:
# {"status":"ok","timestamp":"2026-05-13T12:34:56Z"}

# List tools
curl http://localhost:8080/mcp/resources | jq .

# Create session
curl -X POST http://localhost:8080/sessions \
  -H "Content-Type: application/json" \
  -d '{"user_id":"test_user","ttl_seconds":3600}' | jq .

# Call a tool
curl -X POST http://localhost:8080/mcp/call \
  -H "Content-Type: application/json" \
  -H "X-Session-ID: <session_id>" \
  -d '{
    "jsonrpc":"2.0","id":1,"method":"tools/call",
    "params":{"name":"tool_name","arguments":{}}
  }' | jq .
```

### 6. Stop Development Environment

```bash
# Stop the router (press Ctrl+C in Terminal 1)

# Stop dependencies
docker-compose down redis
```

---

## Docker Deployment

### 1. Build Docker Image

```bash
docker build -f Dockerfile -t mcp-platform:latest .

# Verify image
docker images | grep mcp-platform
```

### 2. Run Single Container

```bash
# Create network
docker network create mcp-net

# Start Redis
docker run -d \
  --name redis \
  --network mcp-net \
  -p 6379:6379 \
  redis:7-alpine

# Start Router
docker run -d \
  --name mcp-platform \
  --network mcp-net \
  -p 8080:8080 \
  --env-file .env \
  -e REDIS_HOST=redis \
  mcp-platform:latest

# Verify
docker logs mcp-platform
curl http://localhost:8080/health
```

### 3. Using Docker Compose

```bash
# Start all services
docker-compose up -d

# Verify services
docker-compose ps

# Check logs
docker-compose logs -f mcp-platform
docker-compose logs -f redis

# Stop services
docker-compose down

# Clean up (remove volumes)
docker-compose down -v
```

### Docker Compose File Structure

```yaml
version: '3.8'

services:
  mcp-platform:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - MCP_ROUTER_ENV=production
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4317
    depends_on:
      - redis
    networks:
      - mcp-net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 5s

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - mcp-net
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "4317:4317"  # OTLP gRPC
      - "16686:16686"  # Jaeger UI
    networks:
      - mcp-net
    environment:
      - COLLECTOR_OTLP_ENABLED=true

volumes:
  redis_data:

networks:
  mcp-net:
    driver: bridge
```

---

## Kubernetes Deployment

### 1. Create Namespace

```bash
kubectl create namespace mcp
kubectl set-context mcp --namespace=mcp
```

### 2. Deploy Redis

```yaml
# redis-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: mcp
data:
  redis.conf: |
    maxmemory 512mb
    maxmemory-policy allkeys-lru

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: mcp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: config
          mountPath: /usr/local/etc/redis
        - name: data
          mountPath: /data
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 10
          periodSeconds: 10
      volumes:
      - name: config
        configMap:
          name: redis-config
      - name: data
        emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: mcp
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
```

Deploy:
```bash
kubectl apply -f redis-deployment.yaml
kubectl get pods -n mcp
```

### 3. Deploy MCP Platform

```yaml
# router-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: router-config
  namespace: mcp
data:
  .env: |
    REDIS_HOST=redis
    REDIS_PORT=6379
    MCP_ROUTER_ENV=production
    MCP_ROUTER_LOG_LEVEL=info
    OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4317
    # Add your MCP servers and REST endpoints here

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-platform
  namespace: mcp
spec:
  replicas: 3  # Three instances for HA
  selector:
    matchLabels:
      app: mcp-platform
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: mcp-platform
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - mcp-platform
              topologyKey: kubernetes.io/hostname
      containers:
      - name: mcp-platform
        image: mcp-platform:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: http
        envFrom:
        - configMapRef:
            name: router-config
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2
        volumeMounts:
        - name: config
          mountPath: /etc/mcp-platform
      volumes:
      - name: config
        configMap:
          name: router-config

---
apiVersion: v1
kind: Service
metadata:
  name: mcp-platform
  namespace: mcp
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: mcp-platform
  sessionAffinity: None  # Stateless
```

Deploy:
```bash
kubectl apply -f router-deployment.yaml
kubectl get pods -n mcp
kubectl get svc -n mcp
```

### 4. Verify Deployment

```bash
# Get service IP
kubectl get svc mcp-platform -n mcp

# Port-forward for testing
kubectl port-forward -n mcp svc/mcp-platform 8080:80

# In another terminal
curl http://localhost:8080/health
```

---

## Configuration

### Environment Variables

```bash
# Server
MCP_ROUTER_PORT=8080
MCP_ROUTER_ENV=production|development|staging
MCP_ROUTER_LOG_LEVEL=debug|info|warn|error

# Redis (Session Store)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0
REDIS_PASSWORD=
REDIS_SSL=false
REDIS_POOL_SIZE=10

# MCP Servers Registration
# Format: server_name:host:port,server_name:host:port
MCP_SERVERS=mcp_server_1:localhost:9000,mcp_server_2:localhost:9001

# REST API Endpoints
# Format: api_name:https://host:port,api_name:https://host:port
REST_ENDPOINTS=weather_api:https://api.weather.com,stock_api:https://api.stocks.com

# OpenTelemetry Tracing
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
OTEL_SERVICE_NAME=mcp-platform
OTEL_EXPORTER_OTLP_HEADERS=

# Logging
LOG_FORMAT=json|text
LOG_LEVEL=debug|info|warn|error

# Rate Limiting
RATE_LIMIT_REQUESTS=1000
RATE_LIMIT_WINDOW_SECONDS=60

# Server Timeouts
HTTP_TIMEOUT_SECONDS=30
BACKEND_CALL_TIMEOUT_SECONDS=10
```

### Configuration File

Alternative to env vars:

```yaml
# config/router.yaml
server:
  port: 8080
  env: production
  log_level: info

redis:
  host: localhost
  port: 6379
  db: 0
  pool_size: 10

mcp_servers:
  - name: weather_service
    endpoint: localhost:9000
    instances: 2
    health_check_interval: 30
  - name: stock_service
    endpoint: localhost:9001
    instances: 1

rest_apis:
  - name: weather_api
    endpoint: https://api.weather.com
    auth:
      type: api_key
      header: X-API-Key
      value: ${WEATHER_API_KEY}
    rate_limit:
      requests: 100
      window_seconds: 60

tracing:
  otlp_endpoint: http://localhost:4317
  service_name: mcp-platform
  sample_rate: 1.0  # 100% for dev, lower for prod

logging:
  format: json
  level: info
  output: stdout|file
  file: /var/log/mcp-platform.log
```

Load:
```bash
export MCP_ROUTER_CONFIG=/etc/mcp-platform/config.yaml
./mcp-platform
```

---

## Monitoring & Observability

### Health Endpoints

```bash
# Liveness (is service running?)
curl http://localhost:8080/health

# Readiness (are dependencies ready?)
curl http://localhost:8080/ready
```

### Logs

View logs:
```bash
# Docker Compose
docker-compose logs -f mcp-platform

# Docker
docker logs -f mcp-platform

# Kubernetes
kubectl logs -f -n mcp deployment/mcp-platform
kubectl logs -f -n mcp deployment/mcp-platform -c mcp-platform
```

Tail recent logs:
```bash
tail -f /var/log/mcp-platform.log | jq .
```

### Tracing (Jaeger)

Access Jaeger UI:
```
http://localhost:16686
```

Query traces:
- Service: mcp-platform
- Operation: tool_call, session_lookup, rate_limit_check
- Filter by trace ID: `X-Trace-ID` from response headers

### Metrics

Prometheus metrics available at:
```
http://localhost:8080/metrics
```

Key metrics:
- `mcp_router_http_requests_total` — Total HTTP requests
- `mcp_router_http_request_duration_seconds` — Request latency
- `mcp_router_tool_calls_total` — Tool invocations
- `mcp_router_tool_call_duration_seconds` — Tool latency
- `mcp_router_redis_operations_total` — Redis operations
- `mcp_router_service_health` — Service health status

Scrape config for Prometheus:
```yaml
scrape_configs:
  - job_name: 'mcp-platform'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/metrics'
```

---

## Scaling & Performance

### Horizontal Scaling

Deploy multiple router instances:
```bash
# Kubernetes
kubectl scale deployment mcp-platform --replicas=5 -n mcp

# Verify
kubectl get pods -n mcp
```

All instances share Redis for session state (stateless).

### Load Balancer Configuration

**Nginx:**
```nginx
upstream mcp_router {
  server mcp-platform-1:8080;
  server mcp-platform-2:8080;
  server mcp-platform-3:8080;
  keepalive 32;
}

server {
  listen 80;
  server_name api.mcp-platform.local;

  location / {
    proxy_pass http://mcp_router;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_set_header X-Trace-ID $request_id;
  }
}
```

**HAProxy:**
```
global
  maxconn 4096

backend mcp_backend
  mode http
  balance roundrobin
  server router1 mcp-platform-1:8080 check
  server router2 mcp-platform-2:8080 check
  server router3 mcp-platform-3:8080 check

frontend mcp_frontend
  bind *:80
  default_backend mcp_backend
```

### Performance Tuning

**Memory:**
- Increase GOMAXPROCS if CPU cores > 4
- Tune Redis pool size (default 10)

**CPU:**
- Run multiple instances (Kubernetes, Docker Swarm)
- Use load balancer to distribute traffic

**Network:**
- Use keep-alive connections (HTTP 1.1)
- Tune read/write buffer sizes
- Enable compression for large responses

**Backend Services:**
- Implement connection pooling in adapter clients
- Add circuit breaker for failing services
- Cache tool schemas (low TTL)

---

## Troubleshooting

### Issue: Cannot connect to Redis

**Symptoms:**
```
[ERROR] Failed to connect to Redis: connection refused
```

**Solutions:**
1. Verify Redis is running:
   ```bash
   docker ps | grep redis
   redis-cli ping
   ```

2. Check Redis connection string:
   ```bash
   echo $REDIS_HOST $REDIS_PORT
   ```

3. Restart Redis:
   ```bash
   docker-compose restart redis
   ```

### Issue: Tool calls timing out

**Symptoms:**
```
[ERROR] Backend call timeout after 10s
```

**Solutions:**
1. Increase timeout in config:
   ```
   BACKEND_CALL_TIMEOUT_SECONDS=30
   ```

2. Check backend service health:
   ```bash
   curl http://localhost:9000/health
   ```

3. Check network latency:
   ```bash
   ping <backend_host>
   ```

### Issue: Rate limiting rejecting all requests

**Symptoms:**
```
HTTP 429 Too Many Requests
```

**Solutions:**
1. Check rate limit config:
   ```bash
   echo $RATE_LIMIT_REQUESTS $RATE_LIMIT_WINDOW_SECONDS
   ```

2. Increase limits if appropriate:
   ```
   RATE_LIMIT_REQUESTS=10000
   RATE_LIMIT_WINDOW_SECONDS=60
   ```

3. Clear session limit state (if stuck):
   ```bash
   redis-cli DEL "ratelimit:*"
   ```

### Issue: High memory usage

**Symptoms:**
```
Container memory exceeded limit
```

**Solutions:**
1. Check active sessions:
   ```bash
   redis-cli DBSIZE
   ```

2. Clean up expired sessions:
   ```bash
   redis-cli SCAN 0 MATCH "session:*"
   ```

3. Increase memory limit in container:
   ```yaml
   resources:
     limits:
       memory: "1Gi"
   ```

### Issue: Traces not appearing in Jaeger

**Symptoms:**
```
No traces in Jaeger UI
```

**Solutions:**
1. Verify Jaeger is running:
   ```bash
   docker ps | grep jaeger
   ```

2. Check OTLP endpoint:
   ```bash
   curl http://localhost:4317
   ```

3. Verify env var:
   ```bash
   echo $OTEL_EXPORTER_OTLP_ENDPOINT
   ```

4. Increase sample rate (if too low):
   ```
   OTEL_TRACES_SAMPLER=always_on
   ```

---

## Backup & Recovery

### Redis Data Backup

```bash
# Manual backup
docker exec redis redis-cli BGSAVE
docker cp redis:/data/dump.rdb ./dump.rdb

# Restore from backup
docker cp ./dump.rdb redis:/data/dump.rdb
docker exec redis redis-cli DEBUG LOADAOF
```

### Session Export

```bash
# Export all sessions
redis-cli --rdb backup.rdb
redis-cli SHUTDOWN SAVE

# Import
redis-cli < backup.rdb
```

---

**Next:** See [docs/DEVELOPMENT.md](DEVELOPMENT.md) for development workflow.
