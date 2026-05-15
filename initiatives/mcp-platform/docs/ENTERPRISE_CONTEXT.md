# Enterprise Deployment Context

## Table of Contents
1. [Current State Architecture](#current-state-architecture)
2. [Target State Architecture](#target-state-architecture)
3. [What Changes and What Stays the Same](#what-changes-and-what-stays-the-same)
4. [Component Glossary](#component-glossary)

---

## Current State Architecture

### Overview

Client requests enter the platform through a layered API gateway stack. All traffic flows through a SaaS-based Virtual Front Door, into Apigee for routing, and then down to OCP-hosted backend services which in turn integrate with Systems of Record.

### Deployment Diagram

```
  External Clients (Browsers, Mobile, 3rd-party)
                       │
                       │  HTTPS
                       ▼
         ┌─────────────────────────┐
         │   VFD — Virtual Front   │
         │   Door  (SaaS)          │
         │  • TLS termination      │
         │  • DDoS protection      │
         │  • Global CDN/WAF       │
         └────────────┬────────────┘
                      │  Proxied request
                      ▼
         ┌─────────────────────────┐
         │   Apigee (APG)          │
         │   Google API Gateway    │
         │  • Auth / OAuth tokens  │
         │  • Path-prefix routing  │
         │  • Rate limiting        │
         │  • API analytics        │
         └────────────┬────────────┘
                      │  Path-prefix → backend mapping
         ┌────────────┼────────────┬────────────┐
         │            │            │            │
         ▼            ▼            ▼            ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐  ┌──────────┐
   │ Service A│ │ Service B│ │ Service C│  │ Service N│
   │  (OCP)   │ │  (OCP)   │ │  (OCP)   │  │  (OCP)   │
   └────┬─────┘ └────┬─────┘ └────┬─────┘  └────┬─────┘
        │             │            │              │
        ▼             ▼            ▼              ▼
   ┌─────────────────────────────────────────────────┐
   │           Systems of Record (SOR)               │
   │  Databases, Mainframes, 3rd-party APIs, ERP     │
   └─────────────────────────────────────────────────┘
```

> **OCP** = OpenShift Container Platform — all backend services are containerized and orchestrated here.

### Request Flow (Current)

| Step | Component | Role |
|------|-----------|------|
| 1 | **Client** | Sends HTTPS REST request |
| 2 | **VFD (Virtual Front Door)** | SaaS ingress — TLS, WAF, CDN, global routing |
| 3 | **Apigee (APG)** | Auth enforcement, path-prefix based routing to the correct OCP service |
| 4 | **OCP Backend Service** | Business logic, data transformation |
| 5 | **SOR (System of Record)** | Authoritative data source — databases, legacy systems, partner APIs |

### Pain Points Addressed by MCP Platform

- AI agents have no standardized interface to discover or invoke capabilities from these backend services.
- Each service has a bespoke REST API; an agent must be custom-integrated with every endpoint.
- No aggregation layer exists — agents must call multiple APIs independently and stitch results together.
- No protocol-level session management for multi-turn AI interactions.

---

## Target State Architecture

### Overview

After the MCP Platform initiative, a new **MCP Gateway** component is introduced between Apigee and the backend services. It acts as a protocol-aware aggregator and router for all MCP traffic — translating, load-balancing, and managing sessions across heterogeneous downstream services.

Internal callers (such as the **Dev Portal**) can reach the MCP Gateway directly without going through Apigee.

### Deployment Diagram

```
  External Clients (Browsers, Mobile, 3rd-party)
                       │
                       │  HTTPS
                       ▼
         ┌─────────────────────────┐
         │   VFD — Virtual Front   │
         │   Door  (SaaS)          │
         │  • TLS termination      │
         │  • DDoS protection      │
         │  • Global CDN/WAF       │
         └────────────┬────────────┘
                      │
                      ▼
         ┌─────────────────────────┐       ┌──────────────────────────┐
         │   Apigee (APG)          │       │   Dev Portal             │
         │   Google API Gateway    │       │   (In-house Developer    │
         │  • Auth / OAuth tokens  │       │    Help Portal)          │
         │  • Path-prefix routing  │       │  • Internal network only │
         │  • Rate limiting        │       │  • Direct MCP access     │
         └────────────┬────────────┘       └────────────┬─────────────┘
                      │ /mcp/* paths                    │ internal calls
                      └─────────────────┬───────────────┘
                                        │
                                        ▼
              ┌─────────────────────────────────────────────┐
              │             MCP Gateway (NEW)               │
              │           deployed on OCP                   │
              │                                             │
              │  ┌─────────────────────────────────────┐   │
              │  │  Transport Layer                     │   │
              │  │  ├─ SSE  : GET /mcp  (persistent)   │   │
              │  │  ├─ HTTP : POST /mcp (session req)   │   │
              │  │  └─ HTTP : POST /mcp/call (stateless)│   │
              │  └─────────────────────────────────────┘   │
              │                    │                        │
              │  ┌─────────────────▼───────────────────┐   │
              │  │  Middleware Pipeline                  │   │
              │  │  Auth • Rate Limit • Session • Trace │   │
              │  └─────────────────┬───────────────────┘   │
              │                    │                        │
              │  ┌─────────────────▼───────────────────┐   │
              │  │  Core Router                         │   │
              │  │  • Namespace resolution              │   │
              │  │  • Service registry lookup           │   │
              │  │  • Response cache (TTL per tool)     │   │
              │  │  ┌───────────────────────────────┐   │   │
              │  │  │  Level-1 Load Balancer         │   │   │
              │  │  │  (across downstream services) │   │   │
              │  │  └───────────────────────────────┘   │   │
              │  │  • Circuit breaker (per service)    │   │
              │  └─────────────────┬───────────────────┘   │
              │                    │                        │
              │  ┌────────────┬────┴──────────┐            │
              │  ▼            ▼               ▼            │
              │ ┌──────┐  ┌──────┐       ┌──────┐         │
              │ │ MCP  │  │ REST │       │ gRPC │         │
              │ │Adapt.│  │Adapt.│       │Adapt.│         │
              │ └──┬───┘  └──┬───┘       └──┬───┘         │
              └────┼─────────┼──────────────┼─────────────┘
                   │         │              │
       ┌───────────┼─────────┼──────────────┼───────────┐
       │           │         │              │    OCP     │
       │           ▼         ▼              ▼           │
       │  ┌─────────────┐ ┌──────────┐ ┌──────────┐    │
       │  │ MCP Server A│ │REST Svc B│ │gRPC Svc C│    │
       │  │  ┌──────────┴─┴──────────┴─┴──────────┤    │
       │  │  │    Level-2 Load Balancer             │    │
       │  │  │  (across instances of same service) │    │
       │  └──┴─────────────────────────────────────┘    │
       └────────────────────────────────────────────────┘
                              │
                              ▼
         ┌─────────────────────────────────────────────────┐
         │           Systems of Record (SOR)               │
         │  Databases, Mainframes, 3rd-party APIs, ERP     │
         └─────────────────────────────────────────────────┘
```

### Request Flow (Target)

| Step | Component | Role |
|------|-----------|------|
| 1 | **Client** | Sends HTTPS request (REST or MCP) |
| 2 | **VFD (Virtual Front Door)** | SaaS ingress — TLS, WAF, CDN, global routing (unchanged) |
| 3a | **Apigee (APG)** | Auth, path-prefix routing — `/mcp/*` paths forwarded to MCP Gateway |
| 3b | **Dev Portal** *(internal only)* | In-house developer portal; calls MCP Gateway directly on the internal network |
| 4 | **MCP Gateway** | Protocol handling, middleware, routing, load balancing, caching |
| 5 | **Downstream Service** | MCP Server, REST API (via adapter), or gRPC service (via adapter) |
| 6 | **SOR (System of Record)** | Authoritative data source — unchanged from current state |

### Two-Level Load Balancing

The MCP Gateway implements load balancing at two distinct levels:

| Level | Scope | Strategy | Purpose |
|-------|-------|----------|---------|
| **Level 1** | Across registered downstream services | Weighted round-robin, least-connections | Distribute tool invocations across all capable services |
| **Level 2** | Across instances of a single service | Round-robin or consistent hashing | Scale individual services horizontally within OCP |

### Ingress Paths Summary

| Caller | Via | Auth Model | Use Case |
|--------|-----|-----------|----------|
| External client / AI agent | VFD → Apigee → MCP Gateway | OAuth2 (Apigee-enforced) | Production traffic from internet-facing agents |
| Dev Portal | Direct → MCP Gateway | Internal service token / mTLS | Developer tooling, API exploration, testing |
| Internal OCP service | Direct → MCP Gateway | Internal service token / mTLS | Service-to-service AI-augmented calls |

---

## What Changes and What Stays the Same

### Unchanged

| Component | Status | Notes |
|-----------|--------|-------|
| VFD (Virtual Front Door) | ✅ Unchanged | Continues to be the global ingress, TLS termination, and WAF |
| Apigee (APG) | ✅ Largely unchanged | Adds new path-prefix rule `/mcp/*` → MCP Gateway; all other routes unaffected |
| OCP Platform | ✅ Unchanged | MCP Gateway is just another OCP-deployed workload |
| Systems of Record | ✅ Unchanged | No SOR modifications required |
| Existing Backend Services | ✅ Unchanged | Continue to exist; some are wrapped behind REST or gRPC adapters by the MCP Gateway |

### New

| Component | Type | Notes |
|-----------|------|-------|
| **MCP Gateway** | New OCP workload | Core routing, protocol translation, LB, circuit breaking, caching |
| **SSE Transport endpoint** | New Apigee route | `GET /mcp` and `POST /mcp` added to Apigee path rules |
| **Dev Portal integration** | New internal route | Dev Portal gains direct network path to MCP Gateway |
| **Service Registry** | New internal store | MCP Gateway maintains its own registry of downstream services |
| **Session Store (Redis)** | New infrastructure | Shared Redis for session state and response cache |

### Modified

| Component | Change |
|-----------|--------|
| **Apigee** | New routing rule: `/mcp/*` → MCP Gateway (all other rules untouched) |
| **Backend services** | Selected services re-registered as MCP tools via REST or gRPC adapter (no code change in most cases) |

---

## Component Glossary

| Term | Full Name | Type | Description |
|------|-----------|------|-------------|
| **VFD** | Virtual Front Door | SaaS | Cloud-based global ingress, CDN, WAF, and TLS termination |
| **APG / Apigee** | Apigee API Gateway | Google SaaS product | API gateway for auth enforcement, path-based routing, rate limiting, and analytics |
| **OCP** | OpenShift Container Platform | On-prem/cloud PaaS | Container orchestration platform where all internal services and the MCP Gateway are deployed |
| **SOR** | System of Record | Internal | Authoritative data stores — relational databases, mainframes, partner APIs, ERP systems |
| **MCP Gateway** | Model Context Protocol Gateway | New OCP workload | Central MCP router/aggregator; this initiative's primary deliverable |
| **Dev Portal** | Developer Help Portal | In-house application | Internal platform for developer documentation, API exploration, and tooling; calls MCP Gateway directly |
| **MCP** | Model Context Protocol | Open standard | Protocol for AI agents to discover and invoke tools from services |
