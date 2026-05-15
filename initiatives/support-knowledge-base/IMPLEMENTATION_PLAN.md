# Implementation Plan — Support Knowledge Base

## Problem Statement

Support agents spend excessive time manually searching across three disconnected systems (Dev Portal, Confluence, support ticket history) to resolve customer issues. This initiative delivers a unified AI-powered knowledge base with a chat interface that dramatically reduces resolution time.

## Approach

Build an end-to-end RAG (Retrieval-Augmented Generation) pipeline:
1. Ingest and embed all source content into a vector database
2. Provide an LLM-powered chat interface backed by semantic retrieval
3. Run continuous sync jobs to keep content fresh

---

## Phase 1 — Foundation & Dev Portal Ingestion

**Goal:** Working ingestion pipeline with vector storage, Dev Portal connected.

- Set up Qdrant vector database + PostgreSQL metadata store
- Build `BaseConnector` interface and document processing pipeline (parse, chunk, embed, upsert)
- Implement Dev Portal connector (full sync)
- Set up Celery + Redis for async job execution
- Build sync run tracking (status, progress, error logging)
- Docker Compose for local development environment

**Exit criteria:** Dev Portal documents fully ingested and queryable via Qdrant API.

---

## Phase 2 — Confluence & Ticket Connectors + Incremental Sync

**Goal:** All three sources ingested; incremental sync running on schedule.

- Implement Confluence connector (full + delta sync using `updated_at`)
- Implement Zendesk/Jira ticket connector (filter: closed/resolved only)
- PII stripping in document processor (email, phone, account IDs)
- Incremental sync logic: content hash deduplication, delta detection
- Celery Beat schedule (per-source cron), distributed lock for overlap prevention
- Weekly full reconciliation job (detect deletions)
- Alerting for sync failures (Slack webhook)

**Exit criteria:** All three sources sync incrementally; zero duplicate vectors on repeated runs.

---

## Phase 3 — RAG Query Engine

**Goal:** Accurate, cited answers to natural language queries.

- Hybrid search: dense (cosine ANN) + sparse (BM25) with RRF merge
- Cross-encoder reranker (`ms-marco-MiniLM-L-6-v2`)
- Context window assembly (deduplicate, truncate to 4k tokens)
- Prompt template with `[Source N]` citation rules
- LLM integration (Azure OpenAI GPT-4o with streaming support)
- Query preprocessing (normalization, optional HyDE expansion)
- Latency target: p95 < 3 seconds end-to-end

**Exit criteria:** Manual evaluation of 50 test queries; ≥ 80% rated "helpful" by team.

---

## Phase 4 — Chat API & UI

**Goal:** Support agents can use the system via a polished web interface.

- FastAPI chat backend: `/chat` (streaming SSE), `/chat/{id}`, `/feedback`, `/search`
- JWT authentication (OIDC integration with org SSO)
- Redis-backed session management (10-turn context window, 4-hour TTL)
- React + TypeScript frontend: conversation pane + sources panel
- Streaming token rendering, source cards with relevance scores, direct links
- Thumbs up/down feedback stored in PostgreSQL
- Session history sidebar (last 30 days)

**Exit criteria:** Deployed to staging; 5 support agents complete UAT with no blocking issues.

---

## Phase 5 — Production Hardening & Observability

**Goal:** Production-ready with monitoring, alerting, and documentation.

- Kubernetes manifests (Deployments, Services, HPA, PodDisruptionBudgets)
- External Secrets Operator integration (Vault / AWS Secrets Manager)
- OpenTelemetry instrumentation across all services
- Grafana dashboards: sync health, query latency, LLM cost, feedback trends
- Alerting rules: sync failures, high latency, unexpected vector count drops
- Load testing: 50 concurrent agents, p95 < 3s
- Runbook documentation
- Security review: PII handling, audit logging, access control

**Exit criteria:** Production deployment approved; all alerts configured; load test passed.

---

## Out of Scope (v1)

- Fine-tuning the LLM on company-specific data
- Real-time webhook-driven sync (v2 consideration)
- Native mobile application
- Multi-language support
- Integration with ticketing system to auto-suggest articles during ticket creation
