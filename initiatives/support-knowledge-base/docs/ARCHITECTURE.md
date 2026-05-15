# Support Knowledge Base — Architecture

## Table of Contents
1. [System Overview](#system-overview)
2. [Ingestion Pipeline](#ingestion-pipeline)
3. [Vector Store Design](#vector-store-design)
4. [RAG Query Engine](#rag-query-engine)
5. [Chat API](#chat-api)
6. [Data Model](#data-model)
7. [Sync State Machine](#sync-state-machine)
8. [Security & Access Control](#security--access-control)
9. [Scalability](#scalability)

---

## System Overview

The Support Knowledge Base is a **Retrieval-Augmented Generation (RAG)** system with three distinct layers:

```
┌──────────────────────────────────────────────────────────────┐
│  INGESTION LAYER                                             │
│  Fetch → Parse → Chunk → Embed → Upsert                     │
│  Runs on schedule; source-isolated connectors                │
└──────────────────────────────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  STORAGE LAYER                                               │
│  Qdrant (vectors + payloads) + PostgreSQL (metadata/state)   │
└──────────────────────────────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  SERVING LAYER                                               │
│  RAG Engine → LLM → Chat API → Chat UI                      │
└──────────────────────────────────────────────────────────────┘
```

---

## Ingestion Pipeline

### Source Connectors

Each source has an isolated connector implementing a common `BaseConnector` interface:

```python
class BaseConnector(ABC):
    @abstractmethod
    def fetch_documents(self, since: datetime | None) -> Iterator[RawDocument]:
        """Yield documents modified since `since`. None = full fetch."""

    @abstractmethod
    def health_check(self) -> bool:
        """Verify the source system is reachable and authenticated."""
```

#### Dev Portal Connector
- **Auth**: API key or OAuth2 client credentials
- **Method**: REST API pagination (`GET /docs?page=N&updated_after=T`)
- **Format**: HTML or Markdown articles
- **Delta detection**: `updated_at` timestamp field

#### Confluence Connector
- **Auth**: Atlassian API token (basic auth over HTTPS)
- **Method**: Confluence REST API v2 (`GET /wiki/api/v2/pages`)
- **Spaces**: Configurable list of space keys to include/exclude
- **Format**: Confluence Storage Format → converted to plain text
- **Delta detection**: `version.when` field + CQL query (`lastModified > "YYYY-MM-DD"`)

#### Support Ticket Connector
- **Zendesk**: REST API (`GET /api/v2/tickets?updated_after=T`, `GET /api/v2/tickets/{id}/comments`)
- **Jira**: REST API (`GET /rest/api/3/search?jql=updated>=-1d`)
- **Extraction strategy**: Combine ticket title + description + resolution comments into one document
- **Filter**: Only ingest tickets in `Resolved` or `Closed` state with a meaningful resolution comment

### Document Processor

```
RawDocument
    │
    ▼
┌───────────────┐
│ HTML/Markup   │  Strip tags, normalize whitespace, extract title
│ Parser        │
└───────┬───────┘
        ▼
┌───────────────┐
│ Chunker       │  Recursive character splitter
│               │  chunk_size=512 tokens, overlap=64 tokens
│               │  Split order: h1 > h2 > paragraph > sentence
└───────┬───────┘
        ▼
┌───────────────┐
│ Deduplicator  │  SHA-256 hash of normalized chunk text
│               │  Skip if hash already in metadata DB
└───────┬───────┘
        ▼
┌───────────────┐
│ Metadata      │  Attach: source_type, doc_id, chunk_index, title,
│ Enricher      │  url, created_at, updated_at, tags, space/product
└───────┬───────┘
        ▼
  EnrichedChunk[]
```

### Embedding Service

- **Primary**: OpenAI `text-embedding-3-small` (1536 dims) — batched at 100 chunks per request
- **Fallback**: `sentence-transformers/all-MiniLM-L6-v2` (384 dims) — self-hosted
- **Batch size**: 100 chunks → single API call
- **Rate limiting**: Exponential backoff with jitter on 429 responses
- **Cost control**: Embeddings are cached by content hash; re-embedding only on content change

---

## Vector Store Design

### Qdrant Collections

**Primary collection: `knowledge_base`**

```json
{
  "name": "knowledge_base",
  "vectors": {
    "size": 1536,
    "distance": "Cosine"
  },
  "sparse_vectors": {
    "bm25": {}
  }
}
```

**Payload schema per vector point:**
```json
{
  "id": "uuid-v4",
  "source_type": "dev_portal | confluence | support_ticket",
  "source_id": "original-doc-id-in-source-system",
  "chunk_index": 3,
  "title": "How to configure SSO",
  "url": "https://devportal.example.com/docs/sso",
  "product": "platform",
  "tags": ["sso", "authentication"],
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-03-20T14:22:00Z",
  "content_hash": "sha256:abc123...",
  "text_preview": "First 200 chars of chunk..."
}
```

### Hybrid Search (Recommended)

Combine dense semantic search with sparse BM25 keyword search using **Reciprocal Rank Fusion (RRF)**:

```
query → dense embedding   → ANN top-50  ──┐
query → BM25 tokenization → BM25 top-50  ──┤ RRF merge → top-20 → rerank → top-5
                                            └──────────────────────────────────────
```

### PostgreSQL Metadata Store

Tracks sync state, document lineage, and agent feedback:

```sql
-- Source document registry
CREATE TABLE documents (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_type  TEXT NOT NULL,          -- dev_portal | confluence | support_ticket
    source_id    TEXT NOT NULL,          -- ID in source system
    title        TEXT,
    url          TEXT,
    content_hash TEXT,
    last_synced  TIMESTAMPTZ,
    updated_at   TIMESTAMPTZ,
    UNIQUE (source_type, source_id)
);

-- Individual chunks mapped to Qdrant point IDs
CREATE TABLE chunks (
    id          UUID PRIMARY KEY,        -- matches Qdrant point ID
    document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
    chunk_index INT,
    content_hash TEXT,
    embedded_at TIMESTAMPTZ
);

-- Sync job runs
CREATE TABLE sync_runs (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_type TEXT,
    started_at  TIMESTAMPTZ DEFAULT now(),
    finished_at TIMESTAMPTZ,
    status      TEXT,                    -- running | success | failed
    docs_fetched INT DEFAULT 0,
    docs_updated INT DEFAULT 0,
    error_msg   TEXT
);

-- Agent feedback on responses
CREATE TABLE response_feedback (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id   TEXT,
    query        TEXT,
    rating       SMALLINT,              -- 1 (helpful) | -1 (not helpful)
    chunk_ids    UUID[],                -- which chunks were shown
    created_at   TIMESTAMPTZ DEFAULT now()
);
```

---

## RAG Query Engine

### Retrieval Flow

```
User query (natural language)
        │
        ▼
Query preprocessing
  - Spell correction
  - Query expansion (HyDE optional)
        │
        ▼
Hybrid search (Qdrant)
  - Dense: cosine ANN (top-50)
  - Sparse: BM25 (top-50)
  - RRF merge → top-20
        │
        ▼
Reranking (cross-encoder)
  - Model: cross-encoder/ms-marco-MiniLM-L-6-v2
  - Re-score top-20, keep top-5
        │
        ▼
Context assembly
  - Deduplicate by document
  - Order by relevance score
  - Truncate to fit context window (max 4,000 tokens of context)
        │
        ▼
Prompt construction
  - System: role + instructions + source citation rules
  - Context: formatted retrieved chunks with [Source N] labels
  - User: original query
        │
        ▼
LLM generation (GPT-4o / Claude)
        │
        ▼
Response + source list returned to Chat API
```

### Prompt Template

```
System:
You are a support knowledge assistant helping support agents resolve customer issues.
Answer ONLY based on the provided context. If the answer is not in the context, say so clearly.
Always cite sources using [Source N] notation. Be concise and actionable.

Context:
[Source 1] (Dev Portal — SSO Configuration Guide)
{chunk_text_1}

[Source 2] (Confluence — Authentication Troubleshooting)
{chunk_text_2}

[Source 3] (Ticket #84291 — Customer: SSO login failing after password reset)
{chunk_text_3}

User question: {query}
```

---

## Chat API

### Endpoints

```
POST   /api/v1/chat                  Start or continue a conversation
GET    /api/v1/chat/{session_id}     Retrieve session history
POST   /api/v1/chat/{session_id}/feedback   Submit thumbs up/down
GET    /api/v1/health                Health check
```

### Chat Request/Response

```json
// POST /api/v1/chat
{
  "session_id": "optional-existing-session-id",
  "query": "Customer cannot log in after enabling SSO, getting 403"
}

// Response
{
  "session_id": "sess_abc123",
  "answer": "This is commonly caused by an incorrect ACS URL in the IdP configuration. [Source 1] Steps to resolve: ...",
  "sources": [
    {
      "source_type": "dev_portal",
      "title": "SSO Configuration Guide",
      "url": "https://devportal.example.com/docs/sso",
      "relevance_score": 0.94
    },
    {
      "source_type": "support_ticket",
      "title": "Ticket #84291 — SSO login failing",
      "url": "https://support.example.com/tickets/84291",
      "relevance_score": 0.87
    }
  ],
  "latency_ms": 1240
}
```

### Session Management

- Sessions stored in Redis (TTL: 4 hours)
- Last 10 exchanges kept for multi-turn conversation context
- Anonymous sessions allowed (no auth required for MVP)

---

## Sync State Machine

```
          ┌──────────────────────────────────────┐
          │           SCHEDULER (Celery Beat)    │
          │  Dev Portal:    every 6 hours        │
          │  Confluence:    every 12 hours       │
          │  Support Tickets: every 1 hour       │
          └──────────────────┬───────────────────┘
                             │ trigger
                             ▼
                     ┌──────────────┐
                     │   PENDING    │
                     └──────┬───────┘
                            │ start
                            ▼
                     ┌──────────────┐
                     │   RUNNING    │◄────── heartbeat updates
                     └──────┬───────┘
                   ┌────────┴────────┐
               success            failure
                   │                 │
                   ▼                 ▼
            ┌──────────┐      ┌──────────────┐
            │ SUCCESS  │      │    FAILED    │
            └──────────┘      └──────┬───────┘
                                     │ retry (max 3)
                                     ▼
                               ┌──────────────┐
                               │   RETRYING   │
                               └──────────────┘
```

---

## Security & Access Control

| Concern | Approach |
|---|---|
| Source credentials | Stored in secrets manager (Vault / AWS Secrets Manager); never in code |
| Chat API auth | JWT tokens issued by SSO; agents must be authenticated |
| Data isolation | Qdrant payload filtering can scope results by `product` or `team` |
| PII in tickets | Ticket connector strips email addresses and phone numbers before ingestion |
| LLM API keys | Injected at runtime via environment variables from secrets manager |
| Audit logging | All queries and responses logged with agent identity for compliance |

---

## Scalability

| Bottleneck | Mitigation |
|---|---|
| Embedding API rate limits | Batch processing + queue with back-pressure |
| Large initial ingest (>100k docs) | Parallelized workers; incremental progress checkpointing |
| Vector search latency | Qdrant HNSW index; horizontal sharding for >5M vectors |
| LLM latency | Streaming responses; response caching for identical queries |
| Sync job overlap | Distributed lock (Redis SETNX) per source prevents concurrent runs |
