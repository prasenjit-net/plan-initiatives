# Data Ingestion & Synchronization

## Overview

The ingestion system is responsible for continuously pulling content from three source systems, transforming it into vector embeddings, and keeping the knowledge base fresh. It is designed for **reliability over speed**: jobs are idempotent, retryable, and checkpointed so a failure never results in data loss or duplication.

---

## Source Systems

### 1. Dev Portal

The Dev Portal is the primary source of official product documentation.

| Property | Value |
|---|---|
| Auth method | API Key (`X-API-Key` header) |
| Base URL | `https://devportal.example.com/api/v1` |
| Content format | Markdown or HTML |
| Pagination | Cursor-based (`next_cursor`) |
| Delta endpoint | `GET /docs?updated_after={ISO8601}&limit=100` |
| Full sync endpoint | `GET /docs?limit=100` |

**Fields extracted:**
- `id` — document ID
- `title` — document title
- `content` — full document body
- `url` — canonical public URL
- `product` — product/section tag
- `updated_at` — last modification timestamp

**Connector pseudocode:**
```python
def fetch_documents(self, since: datetime | None):
    cursor = None
    params = {"limit": 100}
    if since:
        params["updated_after"] = since.isoformat()
    while True:
        if cursor:
            params["cursor"] = cursor
        resp = self.session.get("/docs", params=params)
        resp.raise_for_status()
        data = resp.json()
        for doc in data["items"]:
            yield RawDocument(
                source_type="dev_portal",
                source_id=doc["id"],
                title=doc["title"],
                content=doc["content"],
                url=doc["url"],
                updated_at=doc["updated_at"],
                metadata={"product": doc.get("product")},
            )
        cursor = data.get("next_cursor")
        if not cursor:
            break
```

---

### 2. Confluence

Internal engineering knowledge lives in Confluence spaces.

| Property | Value |
|---|---|
| Auth method | HTTP Basic (`user@example.com:API_TOKEN`) |
| API version | Confluence REST API v2 |
| Base URL | `https://yourorg.atlassian.net/wiki/api/v2` |
| Content format | Confluence Storage Format (XML-like) → stripped to plain text |
| Spaces to index | Configurable via `CONFLUENCE_SPACE_KEYS` env var |
| Delta query | `GET /pages?spaceKey={key}&status=current&sort=modified&cursor={c}` |

**CQL delta query (v1 API fallback):**
```
space = "ENG" AND lastModified >= "2024-01-01" ORDER BY lastModified ASC
```

**Fields extracted:**
- `id` — page ID
- `title` — page title
- `body.storage.value` → strip XML tags to get plain text
- `_links.webui` → canonical URL
- `version.when` → last modified timestamp
- `space.key` → space identifier

**PII stripping:** The processor removes any detected email addresses, phone numbers, and Atlassian user account IDs from extracted text before chunking.

---

### 3. Support Tickets (Zendesk)

Resolved ticket threads represent hard-won institutional knowledge.

| Property | Value |
|---|---|
| Auth method | API Token (Base64 `{email}/token:{api_token}`) |
| Base URL | `https://yourorg.zendesk.com/api/v2` |
| Delta endpoint | `GET /incremental/tickets.json?start_time={unix_ts}` |
| Comment endpoint | `GET /tickets/{id}/comments` |
| Filter | Only `status=closed` or `status=solved` tickets |

**Document construction strategy:**
Each ticket is assembled into a single document:
```
Title: {ticket subject}

PROBLEM:
{ticket description (first message)}

RESOLUTION:
{last agent comment marked as resolution, or last public comment}

TAGS: {ticket tags joined by comma}
PRODUCT: {ticket custom field: product_area}
```

This `(problem, resolution)` framing maximizes relevance during retrieval since support agents phrase queries as problems.

**Jira (alternative):**
- Use JQL: `project = SUP AND status in (Resolved, Done) AND updated >= -1d`
- Extract: `summary`, `description`, resolution comment from `comment.body`

---

## Chunking Strategy

### Algorithm

1. **Structural split first**: Split on markdown/HTML headings (`# h1`, `## h2`, `### h3`)
2. **Paragraph split**: If a section still exceeds `max_chunk_tokens`, split on double newlines
3. **Sentence split**: If a paragraph exceeds limit, split on sentence boundaries
4. **Hard truncate**: As a last resort, split at token limit with overlap

### Parameters

| Parameter | Value | Rationale |
|---|---|---|
| `chunk_size` | 512 tokens | Fits in embedding model context; granular enough for precision |
| `overlap` | 64 tokens | Maintains context across chunk boundaries |
| `min_chunk_size` | 50 tokens | Discard very short chunks (e.g., just a heading) |
| Tokenizer | `tiktoken cl100k_base` | Matches OpenAI embedding model tokenization |

### Special handling for tickets
Ticket documents are treated as a single chunk (they're already short). If they exceed 512 tokens, split on the `RESOLUTION:` boundary, keeping problem+resolution together in the first chunk.

---

## Synchronization Schedule

```
┌─────────────────────────────────────────────────────────────┐
│                   Celery Beat Schedule                      │
├────────────────────┬────────────────┬───────────────────────┤
│ Source             │ Cron           │ Strategy              │
├────────────────────┼────────────────┼───────────────────────┤
│ Dev Portal         │ 0 */6 * * *    │ Incremental (delta)   │
│ Confluence         │ 0 */12 * * *   │ Incremental (delta)   │
│ Support Tickets    │ 0 * * * *      │ Incremental (1h ago)  │
│ Full reconcile     │ 0 2 * * 0      │ Full sync (Sunday 2am)│
└────────────────────┴────────────────┴───────────────────────┘
```

**Full reconcile** runs weekly to detect deletions and content drifts that incremental sync may miss. It computes a diff between source IDs and the metadata DB, then deletes stale vectors from Qdrant.

---

## Idempotency & Deduplication

Every document chunk is identified by a **content hash** (SHA-256 of normalized text). Before upserting:

1. Look up `content_hash` in `chunks` table
2. If hash matches → **skip** (no re-embedding, no Qdrant upsert)
3. If hash differs → **update**: delete old Qdrant point, insert new one
4. If not found → **insert**: embed and insert into Qdrant + metadata DB

This guarantees:
- Re-running the same sync job produces no duplicate vectors
- Unchanged content is never re-embedded (saves cost)
- Content modifications trigger a clean vector replacement

---

## Error Handling

| Error | Behavior |
|---|---|
| Source API 429 (rate limit) | Exponential backoff: 1s, 2s, 4s, 8s (max 4 retries) |
| Source API 5xx | Retry 3 times; mark sync run as `failed` if all retries exhausted |
| Embedding API failure | Retry with backoff; failed chunks queued in `embedding_retry` table |
| Qdrant write failure | Retry 3 times; maintain pending upserts in DB for recovery |
| Partial sync failure | Progress checkpoint saved every 500 documents; resume from checkpoint on retry |

---

## Monitoring & Alerting

Each sync run records:
- `docs_fetched` — total documents pulled from source
- `docs_updated` — documents that changed and were re-embedded
- `docs_skipped` — unchanged documents skipped
- `docs_failed` — documents that failed processing
- `duration_seconds` — total run time

**Alerts (PagerDuty / Slack):**
- Sync run `status = failed` for any source
- `docs_failed > 10%` of `docs_fetched` in a single run
- No successful sync run for a source in > 24 hours
- Qdrant collection size decreases by > 5% (unexpected deletions)

---

## Configuration Reference

```yaml
# ingestion/config.yaml

sources:
  dev_portal:
    base_url: ${DEV_PORTAL_BASE_URL}
    api_key: ${DEV_PORTAL_API_KEY}
    sync_interval_hours: 6
    enabled: true

  confluence:
    base_url: ${CONFLUENCE_BASE_URL}
    username: ${CONFLUENCE_USER}
    api_token: ${CONFLUENCE_API_TOKEN}
    space_keys: ["ENG", "SUPPORT", "PRODUCT"]
    sync_interval_hours: 12
    enabled: true

  zendesk:
    base_url: ${ZENDESK_BASE_URL}
    email: ${ZENDESK_EMAIL}
    api_token: ${ZENDESK_API_TOKEN}
    sync_interval_hours: 1
    enabled: true

embedding:
  provider: openai                         # openai | sentence_transformers
  model: text-embedding-3-small
  dimensions: 1536
  batch_size: 100
  cache_enabled: true

chunking:
  chunk_size_tokens: 512
  overlap_tokens: 64
  min_chunk_tokens: 50

vector_db:
  provider: qdrant
  host: ${QDRANT_HOST}
  port: 6333
  collection: knowledge_base

metadata_db:
  url: ${POSTGRES_URL}
```
