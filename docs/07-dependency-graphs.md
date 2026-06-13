# 07 — Dependency Graphs

Every dependency in the system, drawn. Section 1–4 cover the **whole system** from four
angles (full picture, synchronous HTTP, asynchronous events, infrastructure). Section 5
gives **one graph per service** showing its internal (in-service) structure and every
external dependency it has — callers, callees, events, infrastructure, and third-party
systems.

Legend used throughout:

| Edge style | Meaning |
|---|---|
| `==>` thick arrow | Primary client traffic path |
| `-->` solid arrow | Synchronous HTTP call (internal network, service tokens) |
| `-.->` dashed arrow | Asynchronous event (Redis Streams, at-least-once) |
| cylinder node | Data store owned exclusively by one service |

---

## 1. Whole-system dependency graph

Everything on one canvas: clients, gateway, all 11 services, the event bus, owned
databases, shared infrastructure, and third-party systems.

```mermaid
flowchart TB
    subgraph clients["Clients"]
        web["Next.js Web App"]
        future["Mobile · Telegram · Viber<br/>(future adapters)"]
    end

    gw["gateway :8000"]

    subgraph services["Internal services"]
        id["identity-service :8010"]
        agent["agent-service :8020"]
        mg["model-gateway :8030"]
        kn["knowledge-service :8040"]
        reg["registry-service :8050"]
        doc["document-service :8060"]
        bill["billing-service :8070"]
        intg["integration-service :8080"]
        plat["platform-service :8090"]
        biz["business-service :8100"]
    end

    bus(("Redis Streams<br/>event bus"))

    subgraph infra["Infrastructure"]
        pg[("PostgreSQL<br/>one DB per service")]
        rd[("Redis<br/>cache · arq queues")]
        s3[("S3 / MinIO<br/>object storage")]
    end

    subgraph ext["Third-party systems"]
        llm["Anthropic · OpenAI"]
        stripe["Stripe"]
        google["Google APIs<br/>(OAuth · Drive · Gmail)"]
        brevo["Brevo (email)"]
        tgapi["Telegram Bot API<br/>(ops alerts)"]
        mail["IMAP / SMTP servers"]
        wdav["WebDAV servers"]
        eik["BG Commercial Register<br/>(EIK lookup)"]
    end

    %% client traffic
    web ==> gw
    future ==> gw
    gw -->|"JWKS"| id
    gw ==>|"chat (SSE)"| agent
    gw --> services

    %% service-to-service HTTP
    agent --> mg & kn & reg & biz & doc & bill & intg
    kn --> mg
    kn -->|"file IO"| intg
    doc --> mg
    doc --> reg
    biz --> reg
    biz --> doc
    plat --> id
    plat --> bill

    %% events (condensed — see §3)
    id & mg & kn & biz -.-> bus
    bus -.-> reg & bill & agent & plat

    %% infrastructure
    services --> pg
    services --> rd
    kn & doc --> s3

    %% third-party
    mg --> llm
    bill --> stripe
    id -->|"OAuth login"| google
    id --> eik
    intg --> google & mail & wdav
    plat --> brevo & tgapi

    style agent fill:#fff3d6,stroke:#cc9a06
    style gw fill:#e8f0fe,stroke:#3b6fc9
    style bus fill:#fce8e8,stroke:#c0392b
```

Reading hints:

- **agent-service is the only fan-out hub** (highlighted) — it must reach everything,
  because tools touch everything. Every other service keeps its synchronous fan-out ≤ 2.
- **identity-service and registry-service are leaf services** for synchronous calls —
  they call no other internal service.
- All services share three pieces of infrastructure (own Postgres DB, Redis, and the
  `x7-common` package); only knowledge-service and document-service touch object storage.

---

## 2. Synchronous HTTP dependency graph

Only request/response edges, with what each call is for. An arrow `A --> B` means *A
breaks if B is down* (modulo retries/timeouts) — this is the graph that matters for
deployment ordering and failure-mode analysis.

```mermaid
flowchart LR
    gw["gateway"]
    id["identity-service"]
    agent["agent-service"]
    mg["model-gateway"]
    kn["knowledge-service"]
    reg["registry-service"]
    biz["business-service"]
    doc["document-service"]
    bill["billing-service"]
    intg["integration-service"]
    plat["platform-service"]

    gw -->|"JWKS fetch + cache"| id
    gw ==>|"reverse proxy<br/>(all prefixes)"| agent & id & kn & reg & biz & doc & bill & intg & plat

    agent -->|"complete / embed"| mg
    agent -->|"knowledge_search"| kn
    agent -->|"registry tools"| reg
    agent -->|"invoice / stock /<br/>expense tools"| biz
    agent -->|"document / price /<br/>KSS tools"| doc
    agent -->|"pre-flight balance check"| bill
    agent -->|"gmail / email /<br/>drive tools"| intg

    kn -->|"POST /v1/embed"| mg
    kn -->|"WebDAV / Drive<br/>credentials + file IO"| intg

    doc -->|"AI import / format"| mg
    doc -->|"row data for<br/>document generation"| reg

    biz -->|"counterparty lookup<br/>(canonical roles)"| reg
    biz -->|"invoice PDF render ·<br/>price list reads"| doc

    plat -->|"user / tenant lookups"| id
    plat -->|"usage views (admin)"| bill

    style agent fill:#fff3d6,stroke:#cc9a06
```

**Depth check** — the longest synchronous chain is 3 hops:
`gateway → agent-service → business-service → document-service` (invoice tool that
renders a PDF). No cycles exist; the graph is a DAG.

| Service | Sync fan-out (calls) | Sync fan-in (called by) |
|---|---|---|
| gateway | 1 (identity JWKS) + proxy | clients only |
| identity-service | 0 — leaf | gateway, platform |
| agent-service | 7 — the hub | gateway, future channel adapters |
| model-gateway | 0 internal (LLM APIs only) | agent, knowledge, document |
| knowledge-service | 2 (model-gw, integration) | agent, gateway |
| registry-service | 0 — leaf | agent, business, document, gateway |
| business-service | 2 (registry, document) | agent, gateway |
| document-service | 2 (model-gw, registry) | agent, business, gateway |
| billing-service | 0 internal (Stripe only) | agent, platform, gateway |
| integration-service | 0 internal (external systems only) | agent, knowledge, gateway |
| platform-service | 2 (identity, billing) | gateway |

One dependency is deliberately absent from this matrix: **service-token minting**. Every
service that makes internal calls obtains short-lived tokens from identity-service, but
callers cache tokens until expiry and receivers verify them locally against the
service-token JWKS — so identity-service sits on the token-*renewal* path, never the
per-request path (see [01 §6](./01-architecture-overview.md#6-identity-tenancy-and-authorization)).

---

## 3. Event (asynchronous) dependency graph

Redis Streams topics with consumer groups. Producers publish facts and move on;
consumers react independently and idempotently. An event edge is a *soft* dependency:
if the consumer is down, events queue and are processed on recovery — nothing upstream
breaks.

```mermaid
flowchart LR
    subgraph producers["Producers"]
        id["identity-service"]
        mg["model-gateway"]
        kn["knowledge-service"]
        biz["business-service"]
        any["any service"]
    end

    bus(("Redis Streams"))

    subgraph consumers["Consumers"]
        reg["registry-service"]
        bill["billing-service"]
        agent["agent-service"]
        plat["platform-service"]
    end

    id -.->|"tenant.created ·<br/>user.registered"| bus
    mg -.->|"token.usage"| bus
    kn -.->|"document.ingested"| bus
    biz -.->|"invoice.issued ·<br/>stock.low"| bus
    any -.->|"notification.requested ·<br/>audit.event"| bus

    bus -.->|"tenant.created →<br/>seed system registries"| reg
    bus -.->|"tenant.created → welcome bonus<br/>token.usage → metering"| bill
    bus -.->|"document.ingested →<br/>context cache bust<br/>(sole consumer)"| agent
    bus -.->|"notification.requested · invoice.issued ·<br/>stock.low · audit.event"| plat

    style bus fill:#fce8e8,stroke:#c0392b
```

| Topic | Producer | Consumers | Hard or soft? |
|---|---|---|---|
| `token.usage` | model-gateway | billing-service | Soft — metering catches up after downtime |
| `tenant.created` | identity-service | registry-service, billing-service | Soft — seeding/bonus delayed, not lost |
| `user.registered` | identity-service | none yet (reserved for onboarding/analytics) | Soft |
| `document.ingested` | knowledge-service | agent-service | Soft — cache bust delayed |
| `invoice.issued` | business-service | platform-service | Soft — notification delayed |
| `stock.low` | business-service | platform-service | Soft |
| `notification.requested` | any service | platform-service | Soft — email queue with retry |
| `audit.event` | any service | platform-service | Soft — audit sink |

---

## 4. Infrastructure dependency graph

What each service needs to boot and serve. Every service additionally installs the
`x7-common` shared kernel (config, auth plumbing, pooled httpx, event bus client,
observability) — a build-time dependency, not a runtime hop.

```mermaid
flowchart TB
    subgraph svcs["Services"]
        gw["gateway"]
        id["identity-service"]
        agent["agent-service"]
        mg["model-gateway"]
        kn["knowledge-service"]
        reg["registry-service"]
        biz["business-service"]
        doc["document-service"]
        bill["billing-service"]
        intg["integration-service"]
        plat["platform-service"]
    end

    subgraph pgs["PostgreSQL — database per service"]
        dbid[("identity")]
        dbag[("agent<br/>+ LangGraph checkpoints")]
        dbmg[("modelgw")]
        dbkn[("knowledge<br/>+ pgvector")]
        dbrg[("registry")]
        dbbz[("business")]
        dbdc[("document")]
        dbbl[("billing")]
        dbin[("integration")]
        dbpl[("platform")]
    end

    rd[("Redis 7<br/>cache · rate limits · arq queues ·<br/>event streams · active project")]
    s3[("S3 / MinIO<br/>originals · generated artifacts")]
    common[/"x7-common<br/>shared kernel (build-time)"/]

    gw --> rd
    id --> dbid & rd
    agent --> dbag & rd
    mg --> dbmg & rd
    kn --> dbkn & rd & s3
    reg --> dbrg & rd
    biz --> dbbz & rd
    doc --> dbdc & rd & s3
    bill --> dbbl & rd
    intg --> dbin & rd
    plat --> dbpl & rd

    svcs -.-> common
```

| Service | Own database | Redis usage | Object storage | Third-party |
|---|---|---|---|---|
| gateway | — | rate-limit buckets, JWKS cache | — | — |
| identity-service | `identity` | events, token-cleanup queue | — | Google OAuth, EIK registers |
| agent-service | `agent` (+ checkpoints) | events, retention queues | — | — (everything via services) |
| model-gateway | `modelgw` | events, balance-check queue | — | Anthropic, OpenAI |
| knowledge-service | `knowledge` (pgvector) | events, embed/sync queues, active project | originals | — (Drive/WebDAV via integration-service) |
| registry-service | `registry` | event consumption | — | — |
| business-service | `business` | events, sweep queues | — | — |
| document-service | `document` | import queue | artifacts (signed URLs) | — (Chromium is in-process) |
| billing-service | `billing` | events, top-up/rollup queues | — | Stripe |
| integration-service | `integration` | health-check queue | — | Google, IMAP/SMTP, WebDAV |
| platform-service | `platform` | events, email send queue | — | Brevo, Telegram Bot API |

---

## 5. Per-service dependency graphs

Each graph shows three things at once:

1. **In-service dependencies** (center) — the hexagonal layering inside the service:
   routes/workers → domain services → ports ← adapters. Arrows inside the service box
   follow the fixed dependency direction; adapters *implement* ports (dashed).
2. **Inbound** (left) — who calls this service and why.
3. **Outbound** (right) — internal services, infrastructure, and third-party systems this
   service depends on, plus events produced/consumed.

### 5.1 gateway

No database, no domain logic — pure edge. Its only hard internal dependency is
identity-service's JWKS endpoint (cached, so brief identity downtime does not take the
edge down).

```mermaid
flowchart LR
    clients["Web app ·<br/>future channel adapters"]

    subgraph gw["gateway"]
        direction TB
        mw["middleware chain<br/>CORS · body limits · IP allow-list"]
        authmw["JWT verify (RS256 vs JWKS) ·<br/>membership-claim check"]
        ctx["context headers<br/>inject X-User-Id / X-Company-Id /<br/>X-Roles / X-Request-Id · strip client values"]
        rl["rate limiter<br/>per route class"]
        rt["routing table (config)<br/>path prefix → service"]
        sse["SSE / chunked passthrough"]
        wh["webhook routes<br/>raw-body preservation"]

        mw --> authmw --> ctx --> rl --> rt
        rt --> sse
        rt --> wh
    end

    id["identity-service<br/>(JWKS, cached)"]
    all["all internal services<br/>(reverse proxy)"]
    rd[("Redis<br/>rate-limit buckets")]

    clients ==> mw
    authmw -->|"JWKS fetch on<br/>cache miss / rotation"| id
    rt ==> all
    rl --> rd
```

| Direction | Dependency | Failure impact |
|---|---|---|
| Outbound | identity-service JWKS | Cached — tolerates brief outage; cold start needs it |
| Outbound | Redis | Rate limiting degrades (fail-open or fail-closed by config) |
| Outbound | every service | Per-route: only that prefix 502s |
| Inbound | all clients | Single point of entry — run ≥ 2 replicas first |

### 5.2 identity-service

A deliberate **leaf**: calls no internal service, so the auth path never has a
transitive dependency. Sends OTP/reset emails by publishing events, never by holding
SMTP credentials.

```mermaid
flowchart LR
    subgraph callers["Called by"]
        gw["gateway<br/>(JWKS + all auth flows)"]
        plat["platform-service<br/>(user/tenant lookups)"]
        anysvc["any service<br/>(service-token mint/verify)"]
    end

    subgraph id["identity-service"]
        direction TB
        subgraph edge["edge"]
            r1["routes/auth<br/>register · login · refresh ·<br/>verify · reset · oauth"]
            r2["routes/users · companies ·<br/>memberships"]
            r3["routes/jwks ·<br/>internal/service-token"]
            w1["workers/<br/>expired-token cleanup (daily)"]
        end
        subgraph domain["domain"]
            s1["auth service<br/>Argon2 · RS256 issue · refresh rotation"]
            s2["tenancy service<br/>companies · memberships · roles"]
            s3["impersonation · admin claims"]
            s4["EIK lookup service"]
            p1["ports/<br/>UserRepo · TokenRepo · OAuthProvider ·<br/>EikProvider · EventBus"]
        end
        subgraph adapters["adapters"]
            a1["Postgres repositories"]
            a2["Google OAuth client"]
            a3["EIK provider clients<br/>(per-provider, env-gated)"]
            a4["event publisher"]
        end
        r1 & r2 & r3 --> s1 & s2
        r3 --> s3
        w1 --> s1
        s1 & s2 & s3 & s4 --> p1
        a1 & a2 & a3 & a4 -.implements.-> p1
    end

    db[("identity DB<br/>users · companies · memberships ·<br/>refresh/reset tokens · oauth ·<br/>impersonation")]
    rd[("Redis<br/>events · job queue")]
    goog["Google OAuth<br/>(login only)"]
    eik["BG Commercial Register APIs"]
    bus(("event bus"))

    gw --> r1 & r3
    plat --> r2
    anysvc --> r3
    a1 --> db
    a2 --> goog
    a3 --> eik
    a4 --> rd
    id -.->|"tenant.created · user.registered ·<br/>audit.event · notification.requested<br/>(OTP / reset emails)"| bus
```

| Direction | Dependency | Notes |
|---|---|---|
| Calls (internal) | **none** — leaf service | Keeps the auth path dependency-free |
| Calls (external) | Google OAuth, EIK registers | Login federation; company onboarding lookup |
| Events out | `tenant.created`, `user.registered`, `audit.event`, `notification.requested` | Email delivery is platform-service's job |
| Inbound | gateway (JWKS, auth), platform-service, any service (service tokens) | JWKS consumers cache keys |

### 5.3 agent-service

The **only fan-out hub** in the system — every tool target is a port + httpx adapter so
targets can be flipped (monolith → new service) during migration without touching the
domain. Conversations are an internal module, not a service.

```mermaid
flowchart LR
    subgraph callers["Called by"]
        gw["gateway<br/>(all chat traffic, SSE)"]
        ch["future channel adapters<br/>(Telegram / Viber)"]
    end

    subgraph agent["agent-service"]
        direction TB
        subgraph edge["edge"]
            r1["routes/<br/>agents · chat (SSE) · resume ·<br/>sessions · memory · skills"]
            w1["workers/<br/>temp-skill expiry · trace retention ·<br/>session purge"]
        end
        subgraph runtime["agent runtime (domain)"]
            disc["agent discovery<br/>manifest.yaml scan → registry"]
            runner["LangGraph runner<br/>context load → ReAct loop →<br/>approval interrupt (write tools)"]
            tb["ToolBox<br/>catalog · allow-lists · dispatch"]
            ctxl["context loader<br/>profile · skills · memory · project"]
            mem["memory & skills services"]
        end
        subgraph conv["conversations/ module"]
            cs["ConversationStore port<br/>sessions · messages · attachments ·<br/>summaries · TTL"]
        end
        subgraph ports["ports"]
            p1["LLMPort · RetrievalPort ·<br/>RegistryPort · BusinessPort ·<br/>DocumentPort · IntegrationPort ·<br/>BillingPort"]
        end
        subgraph adapters["adapters"]
            a1["httpx clients<br/>(one per port)"]
            a2["Postgres checkpointer<br/>(LangGraph state)"]
            a3["Postgres conversation repo"]
        end
        r1 --> runner
        w1 --> mem & cs
        runner --> tb & ctxl & cs
        disc --> runner
        tb & ctxl --> p1
        runner --> a2
        a1 -.implements.-> p1
        a3 -.implements.-> cs
    end

    subgraph downstream["Calls (HTTP)"]
        mg["model-gateway<br/>complete / embed"]
        kn["knowledge-service<br/>search / retrieve"]
        reg["registry-service<br/>registry tools"]
        biz["business-service<br/>invoice / stock / expense tools"]
        doc["document-service<br/>document / price / KSS tools"]
        bill["billing-service<br/>pre-flight balance check"]
        intg["integration-service<br/>gmail / email / drive tools"]
    end

    db[("agent DB<br/>checkpoints · sessions · messages ·<br/>attachments · ai_memory · skills ·<br/>tool log · traces")]
    rd[("Redis<br/>events · retention queues")]
    bus(("event bus"))

    gw ==> r1
    ch --> r1
    a1 --> mg & kn & reg & biz & doc & bill & intg
    a2 & a3 --> db
    agent -.->|"audit.event"| bus
    bus -.->|"document.ingested<br/>(context cache bust)"| agent
    w1 --> rd
```

| Direction | Dependency | Notes |
|---|---|---|
| Calls | model-gateway, knowledge, registry, business, document, billing, integration | All behind ports — tool targets are swappable |
| Events out / in | `audit.event` / `document.ingested` | Cache bust is soft |
| Inbound | gateway, future channel adapters | Channels use the chat API, never the store |
| Critical path | model-gateway | A chat turn cannot complete without it |

### 5.4 model-gateway

The only service holding LLM provider keys. Internal fan-out: zero — it only goes
outward to providers. Metering is a structural side effect: every completion emits
`token.usage`, so billing cannot be bypassed.

```mermaid
flowchart LR
    subgraph callers["Called by"]
        agent["agent-service<br/>(chat completions, streaming)"]
        kn["knowledge-service<br/>(embeddings)"]
        doc["document-service<br/>(AI import / format)"]
    end

    subgraph mg["model-gateway"]
        direction TB
        subgraph edge["edge"]
            r1["routes/<br/>v1/complete · v1/embed · v1/models ·<br/>v1/providers (admin) · kill-switch (admin)"]
            w1["workers/<br/>provider balance check (daily)"]
        end
        subgraph domain["domain"]
            comp["completion service<br/>streaming passthrough · retries ·<br/>error normalization"]
            emb["embedding service"]
            meter["metering<br/>token.usage on every call"]
            ks["kill switch<br/>global + per-tenant"]
            preg["provider registry<br/>logical model → provider+model"]
            p1["ports/<br/>LLM · Embedder · ProviderRepo · EventBus"]
        end
        subgraph adapters["adapters"]
            a1["Anthropic adapter"]
            a2["OpenAI adapter"]
            a3["Postgres repo<br/>(encrypted credentials)"]
            a4["event publisher"]
        end
        r1 --> comp & emb
        comp & emb --> ks & meter & preg
        comp & emb & preg --> p1
        w1 --> p1
        a1 & a2 & a3 & a4 -.implements.-> p1
    end

    anthropic["Anthropic API"]
    openai["OpenAI API"]
    db[("modelgw DB<br/>providers · model configs ·<br/>kill-switch state")]
    rd[("Redis<br/>events · job queue")]
    bus(("event bus"))

    agent ==> r1
    kn --> r1
    doc --> r1
    a1 --> anthropic
    a2 --> openai
    a3 --> db
    a4 --> rd
    mg -.->|"token.usage (every call) ·<br/>audit.event ·<br/>notification.requested (balance alert)"| bus
```

| Direction | Dependency | Notes |
|---|---|---|
| Calls (internal) | **none** | Leaf for internal traffic |
| Calls (external) | Anthropic, OpenAI | Provider switch = config change here, invisible to callers |
| Events out | `token.usage`, `audit.event` | Metering is not optional for callers |
| Inbound | agent (hottest path), knowledge, document | Streaming must add near-zero latency |

### 5.5 knowledge-service

The ingestion pipeline is in-service (parse → chunk → embed → index); the two outbound
dependencies are embeddings (model-gateway) and file IO for sync sources
(integration-service). The vector store hides behind a port — pgvector now, swappable.

```mermaid
flowchart LR
    subgraph callers["Called by"]
        agent["agent-service<br/>(knowledge_search tool)"]
        gw["gateway<br/>(library / archive UI)"]
    end

    subgraph kn["knowledge-service"]
        direction TB
        subgraph edge["edge"]
            r1["routes/<br/>files · categories · search · facts ·<br/>projects · sync"]
            w1["workers/<br/>embed-document · sync-webdav ·<br/>sync-drive · sweeps (30 min) · reindex"]
        end
        subgraph domain["domain"]
            ing["ingestion pipeline<br/>parse (PDF/DOCX/XLSX) →<br/>chunk → embed → index"]
            ret["retrieval service<br/>namespace-scoped hybrid search<br/>(vector + keyword)"]
            facts["RAG facts · projects ·<br/>archive permissions"]
            sync["sync engines<br/>WebDAV · Drive — per-file txn,<br/>deadline + continuation"]
            p1["ports/<br/>VectorStore · FileStore ·<br/>EmbedderClient · IntegrationIO · EventBus"]
        end
        subgraph adapters["adapters"]
            a1["pgvector repository"]
            a2["S3 / MinIO client"]
            a3["httpx → model-gateway"]
            a4["httpx → integration-service"]
            a5["event publisher"]
        end
        r1 --> ing & ret & facts & sync
        w1 --> ing & sync
        ing & ret & sync --> p1
        a1 & a2 & a3 & a4 & a5 -.implements.-> p1
    end

    mg["model-gateway<br/>POST /v1/embed"]
    intg["integration-service<br/>WebDAV / Drive credentials + IO"]
    db[("knowledge DB<br/>categories · files · chunks (pgvector) ·<br/>facts · projects · sync state")]
    s3[("S3 / MinIO<br/>original files")]
    rd[("Redis<br/>events · job queues · active project")]
    bus(("event bus"))

    agent ==> r1
    gw --> r1
    a3 --> mg
    a4 --> intg
    a1 --> db
    a2 --> s3
    a5 --> rd
    kn -.->|"document.ingested"| bus
```

| Direction | Dependency | Notes |
|---|---|---|
| Calls | model-gateway (embed), integration-service (sync IO) | Both only on ingestion paths — search itself is self-contained |
| Events out | `document.ingested` | Consumed by agent (cache bust) + platform |
| Inbound | agent-service, gateway | Search is on the agent tool path |
| Infra | pgvector DB, S3, Redis | Only service (with document) needing object storage |

### 5.6 registry-service

A synchronous **leaf** with no jobs — the simplest runtime profile in the system. Its
only asynchronous duty is consuming `tenant.created` to seed system registries.

```mermaid
flowchart LR
    subgraph callers["Called by"]
        agent["agent-service<br/>(registry_query · registry_add_row · …)"]
        biz["business-service<br/>(counterparty lookup)"]
        doc["document-service<br/>(row data for documents)"]
        gw["gateway<br/>(registries / clients / dashboard UI)"]
    end

    subgraph reg["registry-service"]
        direction TB
        subgraph edge["edge"]
            r1["routes/<br/>registries · columns · rows ·<br/>access · templates · clients ·<br/>dashboard"]
            ev["event consumer<br/>tenant.created → seed"]
        end
        subgraph domain["domain"]
            eng["registry engine<br/>typed columns · JSONB rows ·<br/>optimistic locking"]
            acc["access matrix · audit trail ·<br/>row revisions"]
            can["canonical column roles<br/>client_name · eik · offer_number"]
            tpl["template catalog + install<br/>(9 domains × 3 tiers)"]
            cli["clients API · dashboard briefing"]
            p1["ports/<br/>RegistryRepo · AuditRepo · EventBus"]
        end
        subgraph adapters["adapters"]
            a1["Postgres repositories"]
        end
        r1 --> eng & tpl & cli
        eng --> acc & can
        ev --> tpl
        eng & acc & tpl & cli --> p1
        a1 -.implements.-> p1
    end

    db[("registry DB<br/>registries · columns · rows (JSONB) ·<br/>access · audit · revisions · templates")]
    rd[("Redis<br/>event consumption")]
    bus(("event bus"))

    agent ==> r1
    biz --> r1
    doc --> r1
    gw --> r1
    a1 --> db
    ev --> rd
    bus -.->|"tenant.created"| ev
```

| Direction | Dependency | Notes |
|---|---|---|
| Calls (internal) | **none** — leaf | No jobs either; fully synchronous domain |
| Events in | `tenant.created` | Seeds work pipeline, invoices, tasks registries |
| Inbound | agent, business, document, gateway | Highest sync fan-in after the gateway |

### 5.7 business-service

Typed ERP domains with hard invariants. Two outbound dependencies: registry-service for
counterparty resolution and document-service for PDF rendering + price reads. Issued
invoices snapshot counterparty data — so a registry outage never corrupts a legal
document, only blocks new lookups.

```mermaid
flowchart LR
    subgraph callers["Called by"]
        agent["agent-service<br/>(invoice_create · inventory_check ·<br/>stock_move · expense_add · …)"]
        gw["gateway<br/>(business UI)"]
    end

    subgraph biz["business-service"]
        direction TB
        subgraph edge["edge"]
            r1["routes/<br/>invoices (+issue/void/credit-note) ·<br/>items · warehouses · stock ·<br/>expenses · reports"]
            w1["workers/<br/>overdue sweep · low-stock check ·<br/>recurring expenses"]
        end
        subgraph domain["domain"]
            inv["invoicing<br/>sequential numbering · VAT ·<br/>immutability · state machine"]
            stk["inventory<br/>append-only movement ledger ·<br/>materialized levels · reservations"]
            exp["spendings<br/>expenses · recurring · budgets ·<br/>cashflow"]
            p1["ports/<br/>InvoiceRepo · StockRepo · ExpenseRepo ·<br/>CounterpartyLookup · PdfRenderer ·<br/>PriceReader · EventBus"]
        end
        subgraph adapters["adapters"]
            a1["Postgres repositories"]
            a2["httpx → registry-service"]
            a3["httpx → document-service"]
            a4["event publisher"]
        end
        r1 --> inv & stk & exp
        w1 --> inv & stk & exp
        inv & stk & exp --> p1
        a1 & a2 & a3 & a4 -.implements.-> p1
    end

    reg["registry-service<br/>counterparty lookup<br/>(canonical roles)"]
    doc["document-service<br/>invoice PDF render ·<br/>price list reads"]
    db[("business DB<br/>invoices · lines · sequences ·<br/>items · warehouses · movements ·<br/>levels · expenses · budgets")]
    rd[("Redis<br/>events · sweep queues")]
    bus(("event bus"))

    agent ==> r1
    gw --> r1
    a2 --> reg
    a3 --> doc
    a1 --> db
    a4 --> rd
    biz -.->|"invoice.issued · stock.low"| bus
```

| Direction | Dependency | Notes |
|---|---|---|
| Calls | registry-service, document-service | Counterparty IDs + display snapshots stored locally on issue |
| Events out | `invoice.issued`, `stock.low` | Turned into notifications by platform-service |
| Inbound | agent-service (write tools → approval interrupts), gateway | |

### 5.8 document-service

Rendering is isolated here on purpose (Chromium worker pool with hard memory/time
limits). Outbound: model-gateway for AI-assisted imports/formatting and registry-service
for row data feeding document generation.

```mermaid
flowchart LR
    subgraph callers["Called by"]
        agent["agent-service<br/>(generate_document · offer_draft ·<br/>save_margins · KSS tools)"]
        biz["business-service<br/>(invoice PDFs · price reads)"]
        gw["gateway<br/>(templates / prices / margins UI)"]
    end

    subgraph doc["document-service"]
        direction TB
        subgraph edge["edge"]
            r1["routes/<br/>templates · render · generate ·<br/>prices · margins · kss"]
            w1["workers/<br/>price import processing"]
        end
        subgraph domain["domain"]
            tpl["template engine<br/>section JSONB model"]
            rend["render pipeline<br/>template + data → HTML →<br/>PDF / XLSX / DOCX"]
            price["price list<br/>categories · items · history ·<br/>AI-assisted XLSX import"]
            marg["margins<br/>per category/item · access control"]
            kss["KSS<br/>analyze · fill · Excel round-trip"]
            p1["ports/<br/>TemplateRepo · PriceRepo · ArtifactStore ·<br/>PdfEngine · AiFormatter · RegistryData"]
        end
        subgraph adapters["adapters"]
            a1["Postgres repositories"]
            a2["Chromium worker pool<br/>(isolated, hard limits)"]
            a3["S3 / MinIO client<br/>(signed URLs)"]
            a4["httpx → model-gateway"]
            a5["httpx → registry-service"]
        end
        r1 --> tpl & rend & price & marg & kss
        w1 --> price
        tpl & rend & price & marg & kss --> p1
        a1 & a2 & a3 & a4 & a5 -.implements.-> p1
    end

    mg["model-gateway<br/>AI import / format"]
    reg["registry-service<br/>row data for generation"]
    db[("document DB<br/>templates · prices · history ·<br/>imports · margins · access")]
    s3[("S3 / MinIO<br/>generated artifacts")]
    rd[("Redis<br/>import queue")]

    agent ==> r1
    biz --> r1
    gw --> r1
    a4 --> mg
    a5 --> reg
    a1 --> db
    a3 --> s3
    w1 --> rd
```

| Direction | Dependency | Notes |
|---|---|---|
| Calls | model-gateway (AI import), registry-service (row data) | Both optional per endpoint — plain rendering needs neither |
| Inbound | agent, business, gateway | business → doc is on the invoice-issue path |
| Infra | document DB, S3, Redis, in-process Chromium | Artifacts never touch local disk |

### 5.9 billing-service

Listens more than it talks: metering and the welcome bonus arrive as events; the only
outbound calls go to Stripe. The webhook enters through the gateway with raw body
preserved and is signature-verified here.

```mermaid
flowchart LR
    subgraph callers["Called by"]
        agent["agent-service<br/>(pre-flight balance check)"]
        plat["platform-service<br/>(usage views, admin)"]
        gw["gateway<br/>(billing UI · Stripe webhook passthrough)"]
    end

    subgraph bill["billing-service"]
        direction TB
        subgraph edge["edge"]
            r1["routes/<br/>balance · usage (+admin SSE) ·<br/>packages · checkout · auto-topup ·<br/>limit-requests"]
            wh["webhooks/stripe<br/>signature verify · event-ID dedupe"]
            ev["event consumers<br/>token.usage · tenant.created"]
            w1["workers/<br/>auto-top-up check · usage rollups"]
        end
        subgraph domain["domain"]
            led["metering ledger<br/>append-only · event-ID keyed<br/>(idempotent)"]
            bal["balances · limits · alerts"]
            pkg["packages · purchases ·<br/>welcome bonus"]
            top["auto-top-up · saved cards"]
            p1["ports/<br/>LedgerRepo · BalanceRepo ·<br/>PaymentProvider · EventBus"]
        end
        subgraph adapters["adapters"]
            a1["Postgres repositories"]
            a2["Stripe SDK adapter"]
            a3["event publisher"]
        end
        r1 --> bal & pkg & top
        wh --> pkg
        ev --> led & pkg
        w1 --> top & led
        led & bal & pkg & top --> p1
        a1 & a2 & a3 -.implements.-> p1
    end

    stripe["Stripe API<br/>checkout · cards · webhooks"]
    db[("billing DB<br/>balances · usage ledger · pricing ·<br/>packages · purchases · stripe_events ·<br/>auto-topup · limit requests")]
    rd[("Redis<br/>events · job queues")]
    bus(("event bus"))

    agent --> r1
    plat --> r1
    gw --> r1 & wh
    a2 --> stripe
    a1 --> db
    a3 --> rd
    bus -.->|"token.usage (metering) ·<br/>tenant.created (welcome bonus)"| ev
    bill -.->|"notification.requested (alerts) ·<br/>audit.event"| bus
```

| Direction | Dependency | Notes |
|---|---|---|
| Calls (internal) | **none** | Leaf for internal sync traffic |
| Calls (external) | Stripe | Highest-risk code — idempotent webhooks, stored event IDs |
| Events in / out | `token.usage`, `tenant.created` / `notification.requested`, `audit.event` | Ledger keyed by event ID |
| Inbound | agent (balance check), platform, gateway | |

### 5.10 integration-service

The platform's outward arm: every external connectivity concern (OAuth scopes,
IMAP/SMTP, WebDAV) lives behind folder-discovered adapters with a uniform contract.
Calls no internal service.

```mermaid
flowchart LR
    subgraph callers["Called by"]
        agent["agent-service<br/>(gmail_* · email_* · drive tools)"]
        kn["knowledge-service<br/>(sync file IO)"]
        gw["gateway<br/>(integrations UI · OAuth callbacks)"]
    end

    subgraph intg["integration-service"]
        direction TB
        subgraph edge["edge"]
            r1["routes/<br/>catalog · connect/disconnect ·<br/>oauth callbacks · provider verbs"]
            w1["workers/<br/>connection health checks"]
        end
        subgraph domain["domain"]
            cat["catalog + lifecycle<br/>install · connect · access modes"]
            vault["credentials vault<br/>AES-256-GCM, never plaintext out"]
            disc["adapter discovery<br/>folder + manifest (plugin pattern)"]
            p1["ports/<br/>IntegrationAdapter contract<br/>connect · disconnect · read ·<br/>getStatus · provider verbs"]
        end
        subgraph adapters["adapters/ (folder-discovered)"]
            a1["google/<br/>OAuth · Drive IO · Gmail verbs"]
            a2["email/<br/>IMAP/SMTP · send log"]
            a3["webdav/<br/>connection CRUD · browse · file IO"]
            a4["Postgres repositories"]
        end
        r1 --> cat & vault
        w1 --> cat
        cat & disc --> p1
        a1 & a2 & a3 -.implements.-> p1
        vault --> a4
    end

    goog["Google APIs<br/>Drive · Gmail · OAuth"]
    mail["IMAP / SMTP servers"]
    wdav["WebDAV servers"]
    db[("integration DB<br/>connections · credentials (encrypted) ·<br/>email_log · access")]
    rd[("Redis<br/>health-check queue")]

    agent ==> r1
    kn --> r1
    gw --> r1
    a1 --> goog
    a2 --> mail
    a3 --> wdav
    a4 --> db
    w1 --> rd
```

| Direction | Dependency | Notes |
|---|---|---|
| Calls (internal) | **none** | Leaf for internal sync traffic |
| Calls (external) | Google, IMAP/SMTP, WebDAV | Each behind one adapter folder |
| Inbound | agent (email/drive tools), knowledge (sync IO), gateway | |

### 5.11 platform-service

The event sink of the system — four low-traffic modules that mostly consume what other
services publish. Only service holding email (Brevo) and ops-alert (Telegram)
credentials.

```mermaid
flowchart LR
    subgraph callers["Called by"]
        gw["gateway<br/>(bell UI · support UI · /admin zone ·<br/>Brevo webhook passthrough)"]
    end

    subgraph plat["platform-service"]
        direction TB
        subgraph edge["edge"]
            r1["routes/<br/>notifications · preferences ·<br/>support · audit · settings"]
            wh["webhooks/brevo<br/>delivery status"]
            ev["event consumers<br/>notification.requested · invoice.issued ·<br/>stock.low · audit.event"]
            w1["workers/<br/>email send queue (retry/backoff) ·<br/>audit/trace retention"]
        end
        subgraph modules["modules (shared DB, separate tables)"]
            notif["notifications/<br/>bell · preferences · templates (bg/en) ·<br/>delivery tracking"]
            sup["support/<br/>tickets · staff replies"]
            aud["audit/<br/>central sink · retention policies"]
            set["settings/<br/>platform flags · defaults"]
        end
        subgraph ports["ports + adapters"]
            p1["EmailProvider port"]
            a1["Brevo adapter (prod)"]
            a2["log adapter (dev)"]
            a3["Telegram ops-alert adapter"]
            a4["httpx → identity / billing"]
            a5["Postgres repositories"]
        end
        r1 --> notif & sup & aud & set
        wh --> notif
        ev --> notif & aud
        w1 --> notif & aud
        sup -->|"in-process"| notif
        notif --> p1
        a1 & a2 -.implements.-> p1
    end

    idsvc["identity-service<br/>user / tenant lookups"]
    bill["billing-service<br/>usage views (admin)"]
    brevo["Brevo API"]
    tg["Telegram Bot API<br/>(ops channel)"]
    db[("platform DB<br/>notifications · deliveries · preferences ·<br/>tickets · messages · audit_log · settings")]
    rd[("Redis<br/>events · send queue")]
    bus(("event bus"))

    gw --> r1 & wh
    a4 --> idsvc & bill
    a1 --> brevo
    a3 --> tg
    a5 --> db
    w1 --> rd
    bus -.->|"notification.requested ·<br/>invoice.issued · stock.low ·<br/>audit.event"| ev
```

| Direction | Dependency | Notes |
|---|---|---|
| Calls | identity-service (lookups), billing-service (usage views) | Both read-only, admin/UI paths |
| Calls (external) | Brevo, Telegram Bot API | Sole holder of these credentials |
| Events in | `notification.requested`, `invoice.issued`, `stock.low`, `audit.event` | All consumers idempotent (dedupe keys) |
| Inbound | gateway only | Lowest sync fan-in in the system |

---

## 6. Dependency rules (the invariants behind the graphs)

These are the rules the graphs above must keep satisfying as the system grows — review
any new edge against them:

1. **One hub.** Only agent-service may have sync fan-out > 2. A second hub means a
   boundary was drawn wrong.
2. **Leaves stay leaves.** identity-service and registry-service call no internal
   service. The auth path and the data backbone must not acquire transitive
   dependencies. (Service-token minting is the one sanctioned inbound exception for
   identity — and it is cached and locally verified precisely so it never becomes a
   per-request edge.)
3. **No cycles.** The sync HTTP graph is a DAG (§2). If a new feature seems to need
   `A → B → A`, the shared concept belongs in one of them — or in an event.
4. **External systems have exactly one owner.** LLM providers → model-gateway; Stripe →
   billing; Google/IMAP/WebDAV → integration; Brevo/Telegram → platform; OAuth login +
   EIK → identity. No second service may ever hold those credentials.
5. **Databases are private.** No edge may ever point at another service's cylinder.
   Cross-service data moves over HTTP or events only.
6. **Events for facts, HTTP for questions.** If the caller needs an answer to proceed,
   it's a sync call; if it's informing the world, it's an event. Don't disguise
   commands as events.
7. **Inside a service, dependencies point inward.** routes/workers → domain → ports;
   adapters implement ports. A domain module importing httpx or SQLAlchemy is a
   layering bug.
8. **`x7-common` is the only shared code.** Build-time, business-logic-free. Service A
   importing service B's code is forbidden in all cases.

## References

- [01 §3 — system topology](./01-architecture-overview.md#3-system-topology) ·
  [01 §7 — events](./01-architecture-overview.md#7-asynchronous-work-and-events)
- [02 — service catalog + call matrix](./02-service-catalog.md#service-to-service-call-matrix)
- [06 — architectural patterns](./06-architectural-patterns.md)
- Per-service READMEs under [services/](./services/README.md) — each `Dependencies`
  table is the source for the graphs here
