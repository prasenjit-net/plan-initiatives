# Chat Interface Design

## Overview

The chat interface is the primary touchpoint for support agents. It provides a conversational window into the unified knowledge base, surfacing relevant documentation, internal guides, and past ticket resolutions — all cited and grounded.

---

## User Experience

### Core Interaction Flow

```
Agent types query
       │
       ▼
System searches all sources simultaneously
       │
       ▼
LLM generates answer grounded in retrieved context
       │
       ▼
Answer displayed with inline citations [Source 1], [Source 2]
       │
       ▼
Source panel shows clickable cards for each cited document
       │
       ▼
Agent rates the response (👍 / 👎) — optional
```

### UI Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  🔍 Support Knowledge Base                          [Agent Name]│
├──────────────────────────────────┬──────────────────────────────┤
│                                  │  SOURCES                     │
│  💬 CONVERSATION                 │  ─────────────────────────── │
│  ─────────────────────────────   │  [Source 1]                  │
│                                  │  📄 SSO Configuration Guide  │
│  You: Customer can't log in      │  Dev Portal · 94% match      │
│  after enabling SSO, getting     │  devportal.example.com/...   │
│  403 error                       │                              │
│                                  │  [Source 2]                  │
│  Assistant:                      │  🎫 Ticket #84291            │
│  This is commonly caused by      │  SSO login failing after     │
│  an incorrect ACS URL in the     │  password reset              │
│  IdP configuration. [Source 1]   │  support.example.com/...     │
│                                  │                              │
│  Steps to resolve:               │  [Source 3]                  │
│  1. Verify ACS URL matches [S1]  │  📝 Auth Troubleshooting     │
│  2. Check certificate expiry     │  Confluence · ENG space      │
│  3. Confirm SP entity ID [S2]    │  yourorg.atlassian.net/...   │
│                                  │                              │
│  [👍 Helpful]  [👎 Not helpful]  │                              │
│  ─────────────────────────────   │                              │
│                                  │                              │
│  ┌────────────────────────────┐  │                              │
│  │ Ask a follow-up question...│  │                              │
│  └────────────────────────────┘  │                              │
└──────────────────────────────────┴──────────────────────────────┘
```

### Key UI Features

| Feature | Description |
|---|---|
| **Multi-turn conversation** | Session context carries across follow-up questions |
| **Source panel** | Right-hand panel shows cited sources as clickable cards |
| **Source badges** | Color-coded icons for Dev Portal 📄, Confluence 📝, Tickets 🎫 |
| **Relevance score** | Percentage match shown on each source card |
| **Direct links** | "Open in source" button on each card navigates to original |
| **Feedback buttons** | Thumbs up/down per response; stored for retrieval tuning |
| **Copy button** | One-click copy of the answer text |
| **Session history** | Sidebar shows previous conversations (last 30 days) |
| **Streaming** | Answer streams token by token (no waiting for full response) |

---

## Chat API

### Base URL

```
https://kb-api.internal.example.com/api/v1
```

### Authentication

All endpoints require a valid JWT in the `Authorization: Bearer {token}` header. Tokens are issued by the organization's SSO provider (OIDC).

### Endpoints

#### `POST /chat` — Send a message

```json
// Request
{
  "session_id": "sess_abc123",        // optional; omit to start new session
  "query": "Customer cannot log in after enabling SSO, getting 403",
  "filters": {                         // optional: scope search
    "source_types": ["dev_portal", "confluence", "support_ticket"],
    "product": "platform"
  }
}

// Response (streaming: text/event-stream)
data: {"type": "token", "content": "This"}
data: {"type": "token", "content": " is"}
data: {"type": "token", "content": " commonly"}
...
data: {"type": "sources", "sources": [...]}
data: {"type": "done", "session_id": "sess_abc123", "latency_ms": 1240}

// Response (non-streaming: application/json)
{
  "session_id": "sess_abc123",
  "answer": "This is commonly caused by...",
  "sources": [
    {
      "citation_label": "Source 1",
      "source_type": "dev_portal",
      "title": "SSO Configuration Guide",
      "url": "https://devportal.example.com/docs/sso",
      "excerpt": "The ACS URL must match exactly what is configured...",
      "relevance_score": 0.94
    }
  ],
  "latency_ms": 1240
}
```

#### `GET /chat/{session_id}` — Retrieve session history

```json
{
  "session_id": "sess_abc123",
  "created_at": "2024-03-20T09:00:00Z",
  "messages": [
    {"role": "user", "content": "...", "timestamp": "..."},
    {"role": "assistant", "content": "...", "sources": [...], "timestamp": "..."}
  ]
}
```

#### `POST /chat/{session_id}/feedback` — Submit feedback

```json
// Request
{
  "message_index": 1,       // which assistant message (0-indexed)
  "rating": 1,              // 1 = helpful, -1 = not helpful
  "comment": "Missing step about certificate renewal"   // optional
}

// Response
{ "status": "ok" }
```

#### `GET /search` — Direct semantic search (no LLM)

For power users who want raw retrieval results without generation:

```json
// Request query params: ?q=SSO+403+error&top_k=10&source_type=support_ticket

// Response
{
  "results": [
    {
      "source_type": "support_ticket",
      "title": "Ticket #84291",
      "url": "...",
      "text": "...",
      "score": 0.91
    }
  ]
}
```

---

## Frontend Architecture

### Technology Stack

| Layer | Technology |
|---|---|
| Framework | React 18 + TypeScript |
| State management | Zustand |
| Styling | Tailwind CSS |
| Streaming | `EventSource` API (SSE) |
| Auth | OIDC via `@auth0/auth0-react` or equivalent |
| Build | Vite |
| Testing | Vitest + React Testing Library |

### Component Tree

```
<App>
  <AuthProvider>
    <Layout>
      <Sidebar>
        <SessionHistory />
        <NewChatButton />
      </Sidebar>
      <ChatPane>
        <MessageList>
          <UserMessage />
          <AssistantMessage>
            <AnswerText />           // renders citations as clickable refs
            <FeedbackButtons />
          </AssistantMessage>
        </MessageList>
        <InputBar>
          <QueryInput />
          <FilterDropdown />
          <SendButton />
        </InputBar>
      </ChatPane>
      <SourcesPanel>
        <SourceCard />               // one per cited source
      </SourcesPanel>
    </Layout>
  </AuthProvider>
</App>
```

---

## Feedback & Continuous Improvement

Agent feedback drives iterative improvement of the knowledge base:

### Short-term (retrieval tuning)
- Thumbs-down responses trigger a review queue
- Low-rated chunks can be suppressed or down-weighted in reranking
- Patterns of low-rated queries identify coverage gaps

### Medium-term (content gaps)
- Queries with no good results (low max relevance score) are logged
- Weekly report sent to knowledge managers showing "unanswered questions"
- Signals which areas need new documentation or ticket tagging

### Long-term (fine-tuning)
- High-quality (query, answer, source) triples collected from positive feedback
- Used for supervised fine-tuning or preference training of the reranker

---

## Accessibility & Compliance

- WCAG 2.1 AA compliance target
- All source links open in new tab with `rel="noopener noreferrer"`
- Keyboard navigable: Tab through messages, Enter to submit, Esc to cancel
- Screen reader labels on all icon buttons
- Agent identity and all queries/responses are audit-logged for compliance
