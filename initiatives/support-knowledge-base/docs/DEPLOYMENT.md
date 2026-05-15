# Deployment Guide

## Overview

The Support Knowledge Base is containerized and can be deployed via **Docker Compose** (for development/staging) or **Kubernetes** (for production). This guide covers both paths.

---

## Services

| Service | Image | Port | Description |
|---|---|---|---|
| `ingestion` | `kb/ingestion:latest` | — | Celery workers for sync jobs |
| `scheduler` | `kb/ingestion:latest` | — | Celery Beat (cron scheduler) |
| `query-engine` | `kb/query-engine:latest` | 8001 | RAG retrieval + LLM orchestration |
| `chat-api` | `kb/chat-api:latest` | 8000 | FastAPI chat backend |
| `chat-ui` | `kb/chat-ui:latest` | 3000 | React frontend (served via Nginx) |
| `qdrant` | `qdrant/qdrant:v1.9` | 6333/6334 | Vector database |
| `postgres` | `postgres:16` | 5432 | Metadata and sync state DB |
| `redis` | `redis:7-alpine` | 6379 | Session cache + Celery broker |

---

## Environment Variables

Create a `.env` file (never commit this):

```bash
# Source systems
DEV_PORTAL_BASE_URL=https://devportal.example.com/api/v1
DEV_PORTAL_API_KEY=...

CONFLUENCE_BASE_URL=https://yourorg.atlassian.net
CONFLUENCE_USER=service-account@yourorg.com
CONFLUENCE_API_TOKEN=...
CONFLUENCE_SPACE_KEYS=ENG,SUPPORT,PRODUCT

ZENDESK_BASE_URL=https://yourorg.zendesk.com
ZENDESK_EMAIL=service-account@yourorg.com
ZENDESK_API_TOKEN=...

# Embedding & LLM
OPENAI_API_KEY=...
AZURE_OPENAI_ENDPOINT=https://yourorg.openai.azure.com
AZURE_OPENAI_API_KEY=...
AZURE_OPENAI_DEPLOYMENT=gpt-4o

# Infrastructure
POSTGRES_URL=postgresql://kb_user:password@postgres:5432/knowledge_base
QDRANT_HOST=qdrant
REDIS_URL=redis://redis:6379/0

# Auth (JWT verification)
OIDC_ISSUER=https://yourorg.auth0.com/
OIDC_AUDIENCE=https://kb-api.internal.example.com

# Observability
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
```

---

## Docker Compose (Development / Staging)

```yaml
# docker-compose.yml
version: "3.9"

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: knowledge_base
      POSTGRES_USER: kb_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U kb_user"]
      interval: 10s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

  qdrant:
    image: qdrant/qdrant:v1.9.0
    volumes:
      - qdrant_data:/qdrant/storage
    ports:
      - "6333:6333"
    environment:
      QDRANT__SERVICE__GRPC_PORT: "6334"

  ingestion:
    build: ./ingestion
    image: kb/ingestion:latest
    command: celery -A ingestion.tasks worker --loglevel=info --concurrency=4
    env_file: .env
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_started }
      qdrant: { condition: service_started }

  scheduler:
    image: kb/ingestion:latest
    command: celery -A ingestion.tasks beat --loglevel=info --scheduler django_celery_beat.schedulers:DatabaseScheduler
    env_file: .env
    depends_on:
      - ingestion

  query-engine:
    build: ./query-engine
    image: kb/query-engine:latest
    env_file: .env
    ports:
      - "8001:8001"
    depends_on:
      - qdrant
      - postgres

  chat-api:
    build: ./chat-api
    image: kb/chat-api:latest
    env_file: .env
    ports:
      - "8000:8000"
    depends_on:
      - query-engine
      - redis

  chat-ui:
    build: ./chat-ui
    image: kb/chat-ui:latest
    ports:
      - "3000:3000"
    environment:
      VITE_API_BASE_URL: http://localhost:8000

volumes:
  postgres_data:
  redis_data:
  qdrant_data:
```

**Start:**
```bash
docker compose up -d
docker compose logs -f chat-api
```

**Run initial full sync:**
```bash
docker compose exec ingestion python -m ingestion.cli sync --source all --full
```

---

## Kubernetes (Production)

### Recommended Cluster Sizing

| Node Pool | Instance Type | Count | Purpose |
|---|---|---|---|
| General | 4 vCPU / 8 GB | 3 | chat-api, chat-ui, scheduler |
| Memory | 8 vCPU / 32 GB | 2 | ingestion workers, query-engine |
| Vector DB | 8 vCPU / 32 GB | 3 | Qdrant cluster |
| DB | Managed PostgreSQL | — | RDS / CloudSQL |

### Key Manifests

```yaml
# k8s/chat-api/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chat-api
  namespace: knowledge-base
spec:
  replicas: 3
  selector:
    matchLabels:
      app: chat-api
  template:
    spec:
      containers:
        - name: chat-api
          image: kb/chat-api:latest
          ports:
            - containerPort: 8000
          envFrom:
            - secretRef:
                name: kb-secrets
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2"
              memory: "2Gi"
          readinessProbe:
            httpGet:
              path: /api/v1/health
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /api/v1/health
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 30
```

### Secrets Management

Use Kubernetes External Secrets Operator with AWS Secrets Manager or HashiCorp Vault:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: kb-secrets
  namespace: knowledge-base
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: kb-secrets
  data:
    - secretKey: OPENAI_API_KEY
      remoteRef:
        key: knowledge-base/openai
        property: api_key
    - secretKey: CONFLUENCE_API_TOKEN
      remoteRef:
        key: knowledge-base/confluence
        property: api_token
```

---

## Database Migrations

Run migrations with Alembic before starting services:

```bash
# Apply migrations
alembic upgrade head

# Create a new migration after schema changes
alembic revision --autogenerate -m "add feedback table"
```

---

## Initial Data Load

On first deployment, run a full sync for all sources:

```bash
# Trigger via CLI (inside ingestion container)
python -m ingestion.cli sync --source dev_portal --full
python -m ingestion.cli sync --source confluence --full
python -m ingestion.cli sync --source zendesk --full

# Monitor progress
python -m ingestion.cli status
```

Expected times for initial load:
| Source | ~Doc Count | Estimated Time |
|---|---|---|
| Dev Portal | 500–2,000 | 10–30 min |
| Confluence | 1,000–5,000 | 30–90 min |
| Support Tickets | 5,000–50,000 | 1–6 hours |

---

## Observability Stack

| Tool | Purpose |
|---|---|
| **OpenTelemetry Collector** | Receives traces/metrics from all services |
| **Jaeger** | Distributed tracing UI |
| **Prometheus** | Metrics scraping and storage |
| **Grafana** | Dashboards for sync jobs, query latency, LLM costs |
| **Loki** | Log aggregation |

### Key Metrics to Monitor

- `kb_sync_run_duration_seconds` — per-source sync job duration
- `kb_sync_docs_processed_total` — documents processed per run
- `kb_query_latency_seconds` — RAG query end-to-end latency (p50/p95/p99)
- `kb_llm_tokens_total` — LLM token consumption (cost tracking)
- `kb_feedback_rating` — rolling ratio of positive/negative feedback
- `qdrant_collection_vectors_count` — total vectors in store

### Grafana Dashboard Panels (recommended)

1. Sync job health: last run time, success/fail rate per source
2. Query performance: latency histogram, error rate
3. Knowledge base size: vector count over time, by source
4. LLM cost: daily token spend
5. Feedback: positive rate trend, unanswered query volume

---

## Runbooks

### Sync job stuck / not running
```bash
# Check Celery worker status
celery -A ingestion.tasks inspect active

# Check Beat schedule
celery -A ingestion.tasks inspect scheduled

# Force trigger a sync
python -m ingestion.cli sync --source confluence --incremental
```

### Qdrant collection corrupted
```bash
# Check collection health
curl http://qdrant:6333/collections/knowledge_base

# Trigger full rebuild (drops and recreates)
python -m ingestion.cli rebuild --source all
```

### High LLM latency
1. Check LLM provider status page
2. Verify Qdrant query latency separately via `/search` endpoint
3. Check Redis session latency (should be < 5ms)
4. Review reranker performance — consider disabling under load

### Chat API returning empty sources
1. Check Qdrant collection has vectors: `GET /collections/knowledge_base`
2. Verify embedding service is responding
3. Check last sync run status: `SELECT * FROM sync_runs ORDER BY started_at DESC LIMIT 10;`
