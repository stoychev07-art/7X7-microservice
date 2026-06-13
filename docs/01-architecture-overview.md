# 01 — Architecture Overview

## 1. What we are building

An **agentic ERP platform** for SME tenants. The core product surfaces are:

- An AI workspace (streaming chat with tool-using agents) that can read and write business
  data on the user's behalf, with human approval for write actions.
- Configurable business registries (CRM pipeline, invoices, counterparties, assets, …).
- A document layer: library + RAG knowledge base, visual templates, PDF/Excel generation,
  pricing and margins, KSS (construction cost sheets).
- Integrations: Google Workspace (Drive/Gmail), IMAP/SMTP email, WebDAV file servers,
  Stripe-based token billing.

The same backend serves multiple frontends: the Next.js web app today; a mobile app and
Telegram/Viber bots later — all through one gateway, none with privileged access.

## 2. Architectural principles

1. **Single entry point.** All external traffic goes through the API Gateway. Services are
   never exposed publicly; they live on an internal network.
2. **Database-per-service.** Each service owns its data store exclusively. Cross-service
   data is fetched over HTTP or received via events — never by connecting to another
   service's database.
3. **Extensibility by convention, not modification.** New agents, tools, and integration
   adapters are added by dropping files into a discovered folder (manifest + implementation),
   not by editing core runtime code. See [03-agent-platform.md](./03-agent-platform.md).
4. **Hexagonal layering inside every service.** Domain logic depends on ports (Python
   `Protocol`s); concrete adapters (httpx clients, DB drivers, SDKs) are the only place a
   framework or driver is imported, injected at the edge via FastAPI dependencies.
5. **LLM access only through the Model Gateway.** No service holds provider API keys except
   the Model Gateway. This centralizes provider switching, retries, and token metering.
6. **Async-first.** FastAPI + async SQLAlchemy/asyncpg + httpx everywhere; background work
   via per-service workers (arq on Redis); cross-service notifications via Redis Streams.
7. **Tenant isolation everywhere.** Every request carries a tenant (company) context;
   every query in every service is tenant-scoped. Postgres RLS as defense-in-depth in
   services that store tenant data.
8. **Start consolidated, split when forced.** The service catalog defines target boundaries,
   but services may be co-deployed initially (one container, multiple routers) and split
   when scaling or team boundaries demand it. The same rule already shaped the catalog
   itself: boundaries that don't pay for their own deployable are modules, not services
   (see [02 § Deliberately merged](./02-service-catalog.md#deliberately-merged-boundaries)).

## 3. System topology

```mermaid
flowchart TB
    subgraph clients["Clients"]
        web["Next.js Web App"]
        mobile["Mobile App (future)"]
        tg["Telegram Bot Adapter (future)"]
        vb["Viber Bot Adapter (future)"]
    end

    gw["API Gateway<br/>(FastAPI)"]

    subgraph core["Core services (internal network)"]
        idn["identity-service<br/>users · companies · JWT"]
        agent["agent-service<br/>LangGraph agents + tools<br/>sessions · messages"]
        mg["model-gateway<br/>LLM provider abstraction"]
        kn["knowledge-service<br/>ingestion · embeddings · RAG"]
        reg["registry-service<br/>dynamic registries · CRM"]
        biz["business-service<br/>invoices · inventory · spendings"]
        doc["document-service<br/>templates · PDF/Excel · pricing"]
        bill["billing-service<br/>tokens · Stripe"]
        intg["integration-service<br/>Google · email · WebDAV"]
        plat["platform-service<br/>notifications · support ·<br/>audit · settings"]
    end

    subgraph infra["Infrastructure"]
        pg[("PostgreSQL<br/>one DB per service")]
        rd[("Redis<br/>cache · queues · streams")]
        s3[("Object storage<br/>files · artifacts")]
    end

    llm["LLM Providers<br/>Anthropic · OpenAI"]
    ext["External systems<br/>Stripe · Google · Brevo · IMAP · WebDAV"]

    clients --> gw
    gw --> core
    agent --> mg & kn & reg & biz & doc & bill & intg
    mg --> llm
    bill & intg & plat --> ext
    core --> pg
    core --> rd
    kn & doc --> s3
```

### Key flow 1 — a chat turn

Client → gateway → agent-service. The agent loads history from its own conversation store
(sessions and messages live in the agent DB — no network hop on the hottest path), runs
its LangGraph graph (calling tools that hit registry-service, knowledge-service, etc.),
streams tokens back through the gateway via SSE, and persists the turn. Model calls go
through model-gateway, which emits a `token.usage` event that billing-service consumes.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client (Next.js)
    participant GW as Gateway
    participant A as agent-service
    participant MG as model-gateway
    participant T as Tool target<br/>(registry / knowledge / …)
    participant B as billing-service

    C->>GW: POST /agents/{id}/chat (SSE)
    GW->>GW: verify JWT, rate-limit,<br/>inject X-User-Id / X-Company-Id
    GW->>A: forward request
    A->>A: load history (own DB)
    loop LangGraph ReAct loop
        A->>MG: complete(messages, tool specs)
        MG--)B: token.usage event (async)
        MG-->>A: tokens (streamed) / tool calls
        A--)C: SSE token frames (via GW)
        opt model requested a read tool
            A->>T: tool call (HTTP)
            T-->>A: result → appended to messages
        end
    end
    A->>A: persist turn (own DB)
    A--)C: SSE done frame
```

### Key flow 2 — write actions need approval

When an agent proposes a write tool call (create registry row, send email, generate
document), the graph **interrupts**: state is checkpointed to Postgres, the client renders
an approval card, and the graph resumes on confirmation — even after a reconnect.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant A as agent-service
    participant CK as Postgres checkpointer
    participant T as Tool target

    A->>A: model proposes write tool<br/>(kind = "write")
    A->>CK: checkpoint graph state
    A--)C: SSE "approval_required" frame<br/>(tool name + arguments)
    Note over C: user reviews approval card<br/>(can reconnect / switch device)
    C->>A: POST /agents/{id}/resume {approved: true}
    A->>CK: load checkpoint, resume graph
    A->>T: execute write tool
    T-->>A: result
    A--)C: SSE continues to final answer
```

### Key flow 3 — document ingestion & retrieval

Upload or sync (WebDAV/Drive) → knowledge-service stores the file, parses, chunks, embeds
(via model-gateway), and indexes into its vector store. Agents retrieve at query time
through the knowledge-service API — never by touching its DB.

```mermaid
flowchart LR
    up["Upload via UI"] --> kn
    wd["WebDAV sync job"] --> kn
    gd["Drive sync job"] --> kn

    subgraph kn["knowledge-service"]
        store["store original<br/>(S3/MinIO)"] --> parse["parse<br/>PDF / DOCX / XLSX"]
        parse --> chunk["chunk"]
        chunk --> embed["embed"]
        embed --> idx[("pgvector index<br/>per namespace")]
    end

    embed -.->|"POST /v1/embed"| mg["model-gateway"]
    kn -- "document.ingested event" --> bus(("Redis Streams"))

    agent["agent-service<br/>knowledge_search tool"] -->|"POST /search<br/>(namespace-scoped)"| idx
```

## 4. Technology stack

| Concern | Choice | Notes |
|---------|--------|-------|
| Backend services | **Python 3.12 + FastAPI** | Pydantic v2 models as the contract layer |
| Agent orchestration | **LangGraph** | Graphs per agent; Postgres checkpointer for interrupts/resume |
| Frontend | **Next.js (App Router) + TypeScript** | Talks only to the gateway; SSE for streaming |
| Databases | **PostgreSQL 16** (one logical DB per service) | `pgvector` in knowledge-service |
| Cache / queues / events | **Redis 7** | arq for per-service jobs; Redis Streams for cross-service events |
| Object storage | **S3-compatible** (MinIO in dev) | Uploaded files, generated artifacts |
| Auth | **JWT RS256** issued by identity-service | Gateway verifies via JWKS; services trust gateway headers |
| Service-to-service HTTP | httpx (async, pooled) | Internal network only; short-lived service tokens |
| Migrations | Alembic per service | Each service owns its schema history |
| Observability | OpenTelemetry traces + structured JSON logs (structlog) + Sentry | Trace ID propagated from gateway through every hop |
| Packaging / dev | Docker Compose (dev), one image per service | `uv` for dependency management |
| API contracts | OpenAPI per service, auto-generated TS client for the frontend | Generated from FastAPI schemas |

## 5. The API Gateway

The gateway is deliberately thin — it owns *cross-cutting* concerns and nothing
domain-specific:

| Responsibility | Detail |
|----------------|--------|
| Routing | Path-prefix routing table: `/api/v1/auth/* → identity-service`, `/api/v1/agents/* → agent-service`, etc. |
| Authentication | Verifies the JWT signature against identity-service's JWKS; rejects unauthenticated requests (except public routes: login, register, webhooks, health) |
| Context propagation | Injects `X-User-Id`, `X-Company-Id`, `X-Roles`, `X-Request-Id` headers for downstream services; strips any client-supplied values of those headers |
| Rate limiting | Redis-backed buckets per route class (auth, chat, default, webhooks) |
| Streaming | Transparent SSE/chunked passthrough for chat streams |
| CORS, body limits, IP allow-lists for admin routes | |

Webhooks (Stripe, Brevo) pass through the gateway with raw-body preservation and are routed
to the owning service, which performs signature verification itself.

What the gateway does **not** do: business logic, response transformation, aggregation.
If a client needs an aggregate view, the owning service exposes it (e.g. the dashboard
briefing endpoint lives in registry-service).

## 6. Identity, tenancy, and authorization

- **identity-service** is the source of truth for users, companies (tenants), and
  memberships with roles (`owner`, `co-owner`, `admin`, `member`, `viewer`).
- Login issues an RS256 **access token** (short-lived, carries `user_id` and a list of
  company memberships) + a **refresh token** (rotated, stored server-side).
- The active tenant is selected per request via the `X-Company-Id` header; the gateway
  validates membership claims and forwards verified context headers.
- **Role enforcement is local to each service** — services receive the verified role and
  apply their own `require_role(...)` dependencies. Fine-grained grants (per-registry access
  matrices, margin access) stay inside the owning service.
- **Platform admin** (cross-tenant) is a separate claim; admin routes return 404 to
  non-admins (no existence leakage), preserving the monolith's ADR-015 behavior.
- Internal service-to-service calls use short-lived service tokens minted by
  identity-service (client-credentials style), so a stolen internal URL alone is not enough.
  To keep identity-service **off the per-request path**: callers mint once and cache the
  token until shortly before expiry (the `x7-common` helper does this, refreshing in the
  background), and receivers verify tokens **locally** against identity-service's
  service-token JWKS — no verification call per request. A brief identity-service outage
  therefore delays only token *renewal*; traffic with cached tokens continues unaffected.
- **Internal trust model (accepted risk).** Services *extract* user identity from
  gateway-verified headers rather than re-verifying the end-user JWT, and a service token
  proves only that the caller is a platform service — so a compromised internal service
  could forge user context toward its peers. This is accepted because services are never
  publicly reachable, service tokens are audience-scoped, and every write is audited. If
  the risk profile changes, the upgrade path is gateway-signed identity headers (services
  verify the gateway's signature per request) — it slots into `x7_common.auth` without
  touching any domain code.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant GW as Gateway
    participant ID as identity-service
    participant S as Any domain service

    rect rgb(245, 245, 245)
        Note over C,ID: Login (once)
        C->>GW: POST /auth/login (email + password)
        GW->>ID: forward
        ID-->>C: access token (RS256, short-lived)<br/>+ refresh token (rotated)
    end

    rect rgb(245, 245, 245)
        Note over C,S: Every authenticated request
        C->>GW: request + Bearer token + X-Company-Id
        GW->>GW: verify signature against<br/>cached JWKS from identity-service
        GW->>GW: check company membership claim,<br/>strip client-supplied context headers
        GW->>S: forward + verified X-User-Id /<br/>X-Company-Id / X-Roles / X-Request-Id
        S->>S: require_role(...) +<br/>fine-grained grants (own DB)
        S-->>C: response
    end
```

## 7. Asynchronous work and events

Two distinct mechanisms, deliberately not conflated:

1. **Jobs (per-service, private)** — arq workers reading Redis queues owned by the service.
   Examples: knowledge-service embedding jobs and sync sweeps, billing-service auto-top-up
   checks, platform-service email sends, agent-service session retention purges. A
   service's queues are an implementation detail; no other service enqueues into them.
2. **Events (cross-service, published facts)** — Redis Streams topics with consumer groups.
   Producers publish facts; consumers react independently:

| Topic | Producer | Consumers | Purpose |
|-------|----------|-----------|---------|
| `token.usage` | model-gateway | billing-service | Metering every LLM call (feature, agent, tenant, tokens) |
| `document.ingested` | knowledge-service | agent-service (cache bust) | New knowledge available |
| `notification.requested` | any service | platform-service | "Send this user an email / in-app notification" |
| `audit.event` | any service | platform-service | Central audit trail sink |
| `tenant.created` | identity-service | registry-service (seed system registries), billing-service (welcome token bonus) | Tenant onboarding fan-out |
| `user.registered` | identity-service | none yet (reserved for onboarding/analytics) | Registration fact published for future consumers |
| `invoice.issued` | business-service | platform-service | Notify owner; future hooks (accounting export) attach here |
| `stock.low` | business-service | platform-service | Minimum-stock threshold alerts |

Events carry IDs + minimal payload; consumers fetch full data over HTTP if needed
(thin events). Every consumer is idempotent (events may be delivered more than once).

**Durability.** Redis runs with AOF persistence (plus a replica in production) so queued
events survive a process restart. Events that must not be lost relative to a database
write — `token.usage` above all, since it is billing data — additionally go through a
**transactional outbox** in the producing service: the domain row and the pending event
commit in one transaction, and a worker flushes the outbox to the stream. Losing such an
event then requires losing the producer's database, not just a Redis process.

```mermaid
flowchart LR
    id["identity-service"] -- "tenant.created ·<br/>user.registered" --> bus
    mg["model-gateway"] -- "token.usage" --> bus
    kn["knowledge-service"] -- "document.ingested" --> bus
    biz["business-service"] -- "invoice.issued · stock.low" --> bus
    any["any service"] -- "notification.requested<br/>audit.event" --> bus

    bus(("Redis Streams<br/>(consumer groups)"))

    bus -- "tenant.created" --> reg["registry-service<br/>seed system registries"]
    bus -- "tenant.created · token.usage" --> bill["billing-service<br/>welcome bonus · metering"]
    bus -- "document.ingested" --> ag["agent-service<br/>context cache bust"]
    bus -- "notification.requested ·<br/>invoice.issued · stock.low ·<br/>audit.event" --> plat["platform-service<br/>email · in-app · alerts ·<br/>central audit sink"]
```

## 8. Frontend architecture

- **One Next.js app** with two zones: the tenant app (`/`) and the admin SPA (`/admin`).
- Server Components for data-heavy pages (registries, library); client components for the
  chat workspace (SSE streaming, approval cards) and interactive editors.
- A **generated TypeScript API client** from the gateway's merged OpenAPI spec — no
  hand-written fetch wrappers drifting from the backend.
- i18n with the existing `bg`/`en` catalogs carried over.
- Future channels (Telegram/Viber bots, mobile) consume the *same* gateway API. Bot
  adapters are small stateless services that map channel messages onto
  `POST /api/v1/agents/{agent_id}/chat` and stream responses back; they hold a channel
  token ↔ platform user link, nothing more.

## 9. Deployment & environments

- **Dev**: single `docker-compose.yml` — gateway, all services, Postgres, Redis, MinIO,
  plus the Next.js dev server. One command to boot the world.
- **Prod (phase 1)**: same images on a single host via Compose; the architecture does not
  *require* Kubernetes to be correct. Scale-out path: move chat-heavy services
  (agent-service, model-gateway, gateway) to multiple replicas first.
- **Config**: 12-factor environment variables, one `Settings(BaseSettings)` per service
  extending a shared `common.Settings`. Secrets via env/secret manager — never in code.
- **CI**: per-service test + lint + image build, triggered by path filters; contract tests
  validate that each service's OpenAPI spec is backward compatible.

## 10. Cross-cutting standards (every service)

- `GET /health` (liveness) and `GET /ready` (checks DB/Redis) endpoints.
- Structured JSON logs with `request_id`, `user_id`, `company_id`, `service` on every line.
- OpenTelemetry tracing; the gateway starts the trace, services continue it.
- Pydantic-validated request/response models; no raw dicts crossing service boundaries.
- Alembic migrations run on deploy, before the new code serves traffic.
- A `tests/` suite runnable in isolation with a disposable Postgres (testcontainers).
