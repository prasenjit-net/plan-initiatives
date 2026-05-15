# Support Knowledge Base Initiative

## Overview

The Support Knowledge Base is a unified AI-powered knowledge platform that aggregates documentation from disparate sources — Dev Portal, Confluence, and historical support ticket resolutions — into a searchable vector database. Support agents interact with it through a conversational chat interface to rapidly discover solutions to customer problems.

### Problem Statement

Support agents today must manually search across multiple disconnected systems:
- **Dev Portal** — official product documentation and API references
- **Confluence** — internal engineering and process documentation
- **Support Tickets** — institutional knowledge locked inside resolved ticket history (Zendesk, Jira, etc.)

This creates slow resolution times, inconsistent answers, and repeated effort for known problems. A unified, AI-assisted search layer eliminates these gaps.

---

## Initiative Goals

### Primary Objectives

1. **Unified Ingestion Pipeline** — Periodically crawl and ingest content from Dev Portal, Confluence, and support ticket systems into a single vector store
2. **Semantic Search** — Enable natural language queries across all knowledge sources simultaneously
3. **RAG-Powered Chat** — Provide a Retrieval-Augmented Generation (RAG) chat interface that answers agent questions with cited sources
4. **Continuous Sync** — Keep the knowledge base current via scheduled incremental synchronization jobs
5. **Feedback Loop** — Allow agents to rate responses, improving retrieval quality over time

### Success Criteria

- ✓ All three source systems ingested and searchable from a single query
- ✓ Incremental sync runs on a configurable schedule with no data loss
- ✓ Chat interface returns answers with source citations (document title, URL, ticket ID)
- ✓ P95 query response time < 3 seconds end-to-end
- ✓ Agent feedback captured and used to re-rank results
- ✓ System handles >10,000 documents across all sources
- ✓ Deployed and operational with monitoring and alerting

---

## Architecture & Design

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          DATA SOURCES                               │
├─────────────────┬──────────────────┬───────────────────────────────┤
│   Dev Portal    │    Confluence    │    Support Tickets            │
│  (REST/Scrape)  │   (REST API)     │   (Zendesk / Jira API)        │
└────────┬────────┴────────┬─────────┴──────────────┬────────────────┘
         │                 │                          │
         └─────────────────┴──────────────────────────┘
                                   │
                    ┌──────────────▼───────────────┐
                    │      Ingestion Pipeline       │
                    │  ┌────────────────────────┐  │
                    │  │  Connector Layer        │  │
                    │  │  (per-source adapters)  │  │
                    │  └──────────┬─────────────┘  │
                    │  ┌──────────▼─────────────┐  │
                    │  │  Document Processor     │  │
                    │  │  - Parse HTML/Markdown  │  │
                    │  │  - Chunk (512 tokens)   │  │
                    │  │  - Deduplicate          │  │
                    │  └──────────┬─────────────┘  │
                    │  ┌──────────▼─────────────┐  │
                    │  │  Embedding Service      │  │
                    │  │  (OpenAI / local model) │  │
                    │  └──────────┬─────────────┘  │
                    └─────────────┼────────────────┘
                                  │
                    ┌─────────────▼────────────────┐
                    │       Vector Database         │
                    │  (Qdrant / Weaviate / pgvec)  │
                    │  + Relational Metadata Store  │
                    │    (PostgreSQL)               │
                    └─────────────┬────────────────┘
                                  │
                    ┌─────────────▼────────────────┐
                    │       RAG Query Engine        │
                    │  - Semantic retrieval (top-k) │
                    │  - Cross-encoder reranking    │
                    │  - Context window assembly    │
                    │  - Prompt construction        │
                    └─────────────┬────────────────┘
                                  │
                    ┌─────────────▼────────────────┐
                    │         LLM Backend           │
                    │  (Azure OpenAI / Anthropic)   │
                    └─────────────┬────────────────┘
                                  │
                    ┌─────────────▼────────────────┐
                    │   Support Agent Chat UI       │
                    │  - Web application            │
                    │  - Cited sources panel        │
                    │  - Thumbs up/down feedback    │
                    │  - Session history            │
                    └──────────────────────────────┘
```

### Component Summary

| Component | Responsibility |
|---|---|
| **Source Connectors** | Authenticate and pull content from each source system |
| **Document Processor** | Parse, clean, chunk, and deduplicate raw content |
| **Embedding Service** | Convert text chunks into dense vector representations |
| **Vector Database** | Store and index embeddings for ANN (approximate nearest-neighbor) search |
| **Metadata Store** | Track document provenance, sync state, timestamps, and source URLs |
| **RAG Query Engine** | Retrieve relevant chunks and assemble prompts for the LLM |
| **LLM Backend** | Generate natural language answers grounded in retrieved context |
| **Chat API** | REST/WebSocket backend serving the frontend |
| **Chat UI** | Agent-facing web interface with history, citations, and feedback |
| **Sync Scheduler** | Orchestrates periodic incremental sync jobs per source |

---

## Key Design Decisions

### Vector Database: Qdrant (recommended)
Self-hosted, open-source, payload filtering, supports hybrid search (dense + sparse). Alternatively **pgvector** if the team prefers to stay within PostgreSQL.

### Embedding Model
- **OpenAI `text-embedding-3-small`** — strong baseline, hosted, low cost
- **`sentence-transformers/all-MiniLM-L6-v2`** — zero-cost, self-hosted fallback

### Chunking Strategy
- Chunk size: ~512 tokens with 64-token overlap
- Respect document structure: split on headings before splitting on paragraphs
- Ticket resolutions: treat each `(problem, resolution)` pair as one chunk

### Sync Strategy
- **Full sync**: initial load and weekly reconciliation
- **Incremental sync**: delta detection via `updated_at` timestamps or webhook events
- **Deduplication**: content hash + source URL as composite key

---

## Technology Stack

| Layer | Technology |
|---|---|
| Ingestion / ETL | Python (Celery + Beat), Apache Airflow (optional) |
| Embedding | OpenAI API / SentenceTransformers |
| Vector Store | Qdrant |
| Metadata DB | PostgreSQL |
| Cache | Redis |
| LLM | Azure OpenAI GPT-4o |
| Chat API | FastAPI (Python) |
| Chat UI | React + TypeScript |
| Orchestration | Docker Compose / Kubernetes |
| Observability | OpenTelemetry + Grafana + Loki |

---

## Repository Structure

```
support-knowledge-base/
├── README.md                  ← This file
├── IMPLEMENTATION_PLAN.md     ← Phased delivery plan
├── docs/
│   ├── ARCHITECTURE.md        ← Detailed component design
│   ├── DATA_INGESTION.md      ← Connector specs and sync pipeline
│   ├── CHAT_INTERFACE.md      ← Chat UI and API design
│   └── DEPLOYMENT.md          ← Infrastructure and deployment guide
├── ingestion/                 ← Python ETL service
│   ├── connectors/            ← Dev Portal, Confluence, Ticket connectors
│   ├── processors/            ← Chunking, cleaning, embedding
│   └── scheduler/             ← Celery tasks and beat schedule
├── query-engine/              ← RAG retrieval and prompt assembly
├── chat-api/                  ← FastAPI chat backend
├── chat-ui/                   ← React frontend
└── infra/                     ← Docker Compose, K8s manifests, Terraform
```

---

## Phases

| Phase | Description |
|---|---|
| **Phase 1** | Core ingestion pipeline + vector DB + Dev Portal connector |
| **Phase 2** | Confluence and Ticket connectors + incremental sync |
| **Phase 3** | RAG query engine + Chat API |
| **Phase 4** | Chat UI + feedback loop |
| **Phase 5** | Observability, hardening, and production deployment |

See [IMPLEMENTATION_PLAN.md](./IMPLEMENTATION_PLAN.md) for detailed breakdown.
