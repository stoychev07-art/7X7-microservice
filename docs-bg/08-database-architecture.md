# 08 — Архитектура на базите данни

Как се съхраняват данните в платформата: моделът **database-per-service**, какво притежава
всяка база, и — най-важното — как бази, които **никога не могат да споделят таблица или
join**, все пак се реферират логически една към друга чрез HTTP, events и denormalized snapshots.

> **Бележка за статуса.** Всички услуги са 📋 planned. Инвентарите на таблици по-долу са взети
> от секцията *Data owned* на всяка услуга (виж [02 — service catalog](./02-service-catalog.md)
> и per-service READMEs под [services/](./services/README.md)); column-level детайлите са
> **intended/target** schema и са ориентировъчни, докато Alembic baselines не land-нат.

---

## 1. Единственото правило: database per service

Всяка услуга притежава точно една PostgreSQL база. Никоя услуга не се свързва към база на
друга услуга — няма **cross-database foreign keys и няма cross-database joins**. Единственият
начин да се четат данни на друга услуга е през публичното ѝ API (sync HTTP със service token)
или чрез реакция на нейните events.

```mermaid
flowchart TB
    subgraph svcs["Services (own their data)"]
        gw["gateway<br/>(no DB)"]
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

    subgraph pgs["PostgreSQL — one database each"]
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

    id --> dbid
    agent --> dbag
    mg --> dbmg
    kn --> dbkn & s3
    reg --> dbrg
    biz --> dbbz
    doc --> dbdc & s3
    bill --> dbbl
    intg --> dbin
    plat --> dbpl

    gw --> rd
    id --> rd
    agent --> rd
    mg --> rd
    kn --> rd
    reg --> rd
    biz --> rd
    doc --> rd
    bill --> rd
    intg --> rd
    plat --> rd
```

### Защо

- **Independent deployability & blast radius** — schema migration или outage в една
  база не може да corrupt-не или блокира друга услуга.
- **Clear ownership** — един writer на table; всички други четат през API.
- **Polyglot persistence where it pays** — `knowledge` пуска extension-а `pgvector`, а
  `agent` host-ва LangGraph checkpoint tables; останалите остават plain relational.

### Цената (и как я плащаме)

- Няма DB-level referential integrity между услуги → integrity се enforce-ва от owning
  service и от **idempotent, eventually-consistent** event consumers.
- Няма joins между услуги → или викате API-то на owner-а, или държите **denormalized
  snapshot** на малкото полета, които ви трябват (виж invoices ↔ counterparties по-долу).

---

## 2. Cross-cutting conventions

Тези правила важат за всяка база, освен ако не е отбелязано друго.

| Convention | Detail |
|---|---|
| **Tenant scoping** | Почти всяка business table носи `company_id` (tenant-а). Това е *logical* reference към `identity.companies.id` — никога SQL FK. Postgres **Row-Level Security (RLS)** policies enforce-ват tenant isolation (посочено в registry/agent checklists). |
| **User reference** | User-owned rows носят `user_id`, logical reference към `identity.users.id`. |
| **Primary keys** | UUIDs (безопасни за mint-ване client-side, без cross-service sequence coupling). Изключението е **legal invoice numbering**, което използва per-tenant sequence rows за gap-free numbers. |
| **Encryption at rest (field level)** | Secrets са AES-256-GCM encrypted in-column: `identity` external tokens, `modelgw.providers` credentials, `integration.credentials`. |
| **Migrations** | Alembic per service; всяка услуга има собствена migration history. |
| **Idempotency** | Event-fed tables dedupe-ват по event/idempotency key (`billing.usage_ledger` по event ID, `billing.stripe_events` по Stripe event ID, notifications по payload dedupe key). |
| **Object storage** | Големи binaries никога не живеят в Postgres — originals и generated artifacts отиват в S3/MinIO; базата пази само keys/metadata. |

---

## 3. Database catalog

| Database | Owning service | Port | Extensions / notable | Stores |
|---|---|---|---|---|
| `identity` | identity-service | 8010 | RS256 keypair (JWKS) | users, companies, memberships, tokens |
| `agent` | agent-service | 8020 | LangGraph checkpoint tables | sessions, messages, memory, skills, traces |
| `modelgw` | model-gateway | 8030 | AES-GCM credentials | providers, model configs, kill switch |
| `knowledge` | knowledge-service | 8040 | **pgvector** | documents, chunks+embeddings, facts, sync state |
| `registry` | registry-service | 8050 | JSONB rows, RLS | dynamic registries, templates, audit |
| `document` | document-service | 8060 | JSONB templates | templates, price list, margins |
| `billing` | billing-service | 8070 | append-only ledger | balances, usage ledger, Stripe data |
| `integration` | integration-service | 8080 | AES-GCM credentials | connections, credentials vault, email log |
| `platform` | platform-service | 8090 | — | notifications, support, audit sink, settings |
| `business` | business-service | 8100 | per-tenant sequences | invoices, inventory, expenses |
| — | gateway | 8000 | **no database** | (Redis only: rate limits, JWKS cache) |

---

## 4. Per-database schemas

Всяка диаграма показва **intra-database** relationships (реални FKs вътре в тази DB). Columns
със suffix `_ref` или анотирани *"logical → …"* са cross-service references, които **не са**
SQL foreign keys (resolve-ват се според §5).

### 4.1 `identity` — users, tenants, trust

Source of truth, към който в крайна сметка сочи `company_id` / `user_id` на всяка друга база.

```mermaid
erDiagram
    users ||--o{ user_companies : "member of"
    companies ||--o{ user_companies : "has member"
    users ||--o{ refresh_tokens : "owns"
    users ||--o{ password_reset_tokens : "requests"
    users ||--o{ oauth_identities : "linked"
    users ||--o{ impersonation_sessions : "admin / target"

    users {
        uuid id PK
        string email
        string password_hash "Argon2"
        bool is_platform_admin "cross-tenant claim"
    }
    companies {
        uuid id PK
        string name
        string eik "BG company id"
        jsonb branding
    }
    user_companies {
        uuid user_id FK
        uuid company_id FK
        enum role "owner|co-owner|admin|member|viewer"
    }
    refresh_tokens {
        uuid id PK
        uuid user_id FK
        timestamp expires_at
    }
    password_reset_tokens {
        uuid id PK
        uuid user_id FK
    }
    oauth_identities {
        uuid id PK
        uuid user_id FK
        string provider "google"
        string external_id
    }
    impersonation_sessions {
        uuid id PK
        uuid admin_user_id FK
        uuid target_user_id FK
        uuid company_id
    }
```

`user_companies` е membership/role join-ът, който захранва `X-Roles` header-а и цялата
tenant authorization (виж [01 §6](./01-architecture-overview.md#6-identity-tenancy-and-authorization)).

### 4.2 `agent` — conversations & agent runtime

```mermaid
erDiagram
    sessions ||--o{ messages : "contains"
    messages ||--o{ attachments : "has"
    sessions ||--o{ tool_validation_log : "logs"
    sessions ||--o{ ai_traces : "traces"

    sessions {
        uuid id PK
        uuid user_id "logical → identity.users"
        uuid company_id "logical → identity.companies"
        string agent_id "manifest id"
        string channel "web|telegram|…"
    }
    messages {
        uuid id PK
        uuid session_id FK
        enum role "user|assistant|tool"
        text content
        string tool_call_id
    }
    attachments {
        uuid id PK
        uuid message_id FK
        string s3_key "→ S3/MinIO"
    }
    ai_memory {
        uuid id PK
        uuid user_id "logical → identity"
        uuid company_id "logical → identity"
        text content
    }
    user_skills {
        uuid id PK
        uuid user_id "logical → identity"
        text snippet
        timestamp expires_at
    }
    tool_validation_log {
        uuid id PK
        uuid session_id FK
    }
    ai_traces {
        uuid id PK
        uuid session_id FK
    }
```

Плюс **LangGraph checkpoint tables** (managed by the Postgres checkpointer), keyed by thread id,
aligned with `sessions` — те пазят interrupt/resume state за write-approval pauses.

### 4.3 `modelgw` — provider registry

```mermaid
erDiagram
    providers ||--o{ model_configs : "exposes"
    providers {
        uuid id PK
        string name "anthropic|openai"
        bytea credentials "AES-256-GCM"
    }
    model_configs {
        uuid id PK
        uuid provider_id FK
        string logical_name "caller-facing model name"
        string provider_model
    }
    kill_switch_state {
        string scope "global|tenant"
        uuid company_id "logical → identity (nullable)"
        bool enabled
    }
```

### 4.4 `knowledge` — library, vectors, sync

```mermaid
erDiagram
    doc_categories ||--o{ doc_files : "groups"
    doc_files ||--o{ document_chunks : "chunked into"
    sync_folders ||--o{ sync_state : "tracks"

    doc_categories {
        uuid id PK
        uuid company_id "logical → identity"
        string name
    }
    doc_files {
        uuid id PK
        uuid category_id FK
        uuid company_id "logical → identity"
        string namespace "retrieval scope"
        string s3_key "→ S3/MinIO original"
    }
    document_chunks {
        uuid id PK
        uuid file_id FK
        string namespace
        vector embedding "pgvector"
        text content
    }
    rag_facts {
        uuid id PK
        uuid company_id "logical → identity"
        vector embedding "pgvector"
        text content
    }
    projects {
        uuid id PK
        uuid company_id "logical → identity"
        uuid user_id "logical → identity"
        string name
    }
    sync_folders {
        uuid id PK
        uuid company_id "logical → identity"
        string provider "webdav|drive"
        uuid connection_ref "logical → integration.connections"
    }
    sync_state {
        uuid id PK
        uuid folder_id FK
        string file_path
        string status
    }
```

Active project per user се кешира в **Redis**, не се пази тук.

### 4.5 `registry` — dynamic tables

```mermaid
erDiagram
    registries ||--o{ registry_columns : "defines"
    registries ||--o{ registry_rows : "holds"
    registries ||--o{ registry_access : "controls"
    registries ||--o{ registry_audit : "audits"
    registry_rows ||--o{ row_revisions : "versions"
    registry_templates ||--o{ template_columns : "defines"

    registries {
        uuid id PK
        uuid company_id "logical → identity"
        string name
    }
    registry_columns {
        uuid id PK
        uuid registry_id FK
        string name
        string type
        string canonical_role "client_name|eik|offer_number|…"
    }
    registry_rows {
        uuid id PK
        uuid registry_id FK
        jsonb values
        int version "optimistic lock"
    }
    registry_access {
        uuid id PK
        uuid registry_id FK
        uuid user_id "logical → identity"
        enum role
    }
    registry_audit {
        uuid id PK
        uuid registry_id FK
        uuid user_id "logical → identity"
        string action
    }
    row_revisions {
        uuid id PK
        uuid row_id FK
        jsonb snapshot
    }
    registry_templates {
        string slug PK
        string tier "core|standard|pro"
    }
    template_columns {
        uuid id PK
        string template_slug FK
        string canonical_role
    }
```

**Canonical column roles** са semantic glue, което позволява на други услуги (и agents)
да намерят например counterparty по `client_name`/`eik` през различно именувани tenant tables.

### 4.6 `document` — templates, price list, margins

```mermaid
erDiagram
    price_categories ||--o{ price_items : "groups"
    price_items ||--o{ price_history : "tracks"
    price_categories ||--o{ category_margins : "margin"
    price_items ||--o{ item_margins : "margin"

    document_templates {
        uuid id PK
        uuid company_id "logical → identity"
        jsonb sections "visual editor model"
    }
    price_categories {
        uuid id PK
        uuid company_id "logical → identity"
        string name
    }
    price_items {
        uuid id PK
        uuid category_id FK
        string name
        numeric price
    }
    price_history {
        uuid id PK
        uuid item_id FK
        numeric price
        timestamp changed_at
    }
    price_imports {
        uuid id PK
        uuid company_id "logical → identity"
        string status
    }
    category_margins {
        uuid id PK
        uuid category_id FK
        numeric margin
    }
    item_margins {
        uuid id PK
        uuid item_id FK
        numeric margin
    }
    margin_access {
        uuid id PK
        uuid user_id "logical → identity"
        enum role
    }
```

### 4.7 `billing` — token economy & payments

```mermaid
erDiagram
    packages ||--o{ purchases : "bought as"

    balances {
        uuid company_id "logical → identity"
        uuid user_id "logical → identity (nullable)"
        bigint token_balance
    }
    usage_ledger {
        uuid id PK
        uuid company_id "logical → identity"
        string event_id "UNIQUE — idempotency"
        string feature
        string agent_id "attribution"
        bigint tokens
    }
    token_pricing {
        uuid id PK
        string model
        numeric price_per_1k
    }
    packages {
        uuid id PK
        string name
        bigint tokens
        numeric price
    }
    purchases {
        uuid id PK
        uuid company_id "logical → identity"
        uuid package_id FK
        string stripe_session_id
    }
    stripe_events {
        string event_id PK "Stripe id — dedupe"
        string type
    }
    auto_topup_settings {
        uuid company_id "logical → identity"
        bigint threshold
    }
    auto_topup_log {
        uuid id PK
        uuid company_id "logical → identity"
    }
    limit_requests {
        uuid id PK
        uuid company_id "logical → identity"
        uuid user_id "logical → identity"
        string status
    }
    bonus_settings {
        uuid id PK
        bigint welcome_tokens
    }
```

`usage_ledger` е **append-only** и keyed by `token.usage` event ID, така че at-least-once
delivery никога не double-bill-ва.

### 4.8 `integration` — connections & credential vault

```mermaid
erDiagram
    connections ||--o{ credentials : "secured by"
    connections ||--o{ email_log : "logs"

    connections {
        uuid id PK
        uuid company_id "logical → identity"
        uuid user_id "logical → identity"
        string provider "google|email|webdav"
        string status
    }
    credentials {
        uuid id PK
        uuid connection_id FK
        bytea secret "AES-256-GCM"
    }
    email_log {
        uuid id PK
        uuid connection_id FK
        enum direction "in|out"
        timestamp at
    }
    integration_access {
        uuid id PK
        string provider
        enum mode "all|excluded|exclusive"
    }
```

`connections` rows са target-ът на `knowledge.sync_folders.connection_ref`.

### 4.9 `platform` — notifications, support, audit, settings

```mermaid
erDiagram
    notifications ||--o{ deliveries : "sent via"
    support_tickets ||--o{ support_messages : "thread"

    notifications {
        uuid id PK
        uuid company_id "logical → identity"
        uuid user_id "logical → identity"
        string type
        bool read
    }
    deliveries {
        uuid id PK
        uuid notification_id FK
        string channel "email|in-app|telegram"
        string status "Brevo webhook"
    }
    preferences {
        uuid id PK
        uuid user_id "logical → identity"
        jsonb settings
    }
    support_tickets {
        uuid id PK
        uuid company_id "logical → identity"
        uuid user_id "logical → identity"
        string status
    }
    support_messages {
        uuid id PK
        uuid ticket_id FK
        uuid author_id "logical → identity"
        text body
    }
    audit_log {
        uuid id PK
        uuid company_id "logical → identity"
        uuid user_id "logical → identity"
        string service "source service"
        string action
    }
    platform_settings {
        string key PK
        jsonb value
    }
```

`audit_log` е **central sink** — всяка услуга пише тук индиректно чрез publish на
`audit.event`; rows reference-ват users/companies/actions през всички услуги логически.

### 4.10 `business` — invoicing, inventory, spendings

```mermaid
erDiagram
    invoices ||--o{ invoice_lines : "has"
    items ||--o{ invoice_lines : "billed as"
    items ||--o{ stock_movements : "moves"
    warehouses ||--o{ stock_movements : "location"
    items ||--o{ stock_levels : "level"
    warehouses ||--o{ stock_levels : "level"
    expense_categories ||--o{ expenses : "categorizes"
    expense_categories ||--o{ budgets : "budgeted"

    invoices {
        uuid id PK
        uuid company_id "logical → identity"
        uuid counterparty_id "logical → registry row"
        jsonb counterparty_snapshot "denormalized at issue"
        string number "per-tenant sequence"
        enum status "draft|issued|paid|overdue|void"
    }
    invoice_lines {
        uuid id PK
        uuid invoice_id FK
        uuid item_id FK
        numeric qty
        numeric unit_price "read from document price list"
    }
    invoice_sequences {
        uuid company_id "logical → identity"
        string series
        bigint next_number "gap-free, FOR UPDATE"
    }
    items {
        uuid id PK
        uuid company_id "logical → identity"
        string sku
        string name
    }
    warehouses {
        uuid id PK
        uuid company_id "logical → identity"
        string name
    }
    stock_movements {
        uuid id PK
        uuid item_id FK
        uuid warehouse_id FK
        enum type "receipt|issue|transfer|adjust"
        numeric qty "append-only ledger"
    }
    stock_levels {
        uuid item_id FK
        uuid warehouse_id FK
        numeric qty "materialized, never < 0"
    }
    expenses {
        uuid id PK
        uuid company_id "logical → identity"
        uuid counterparty_id "logical → registry row"
        uuid category_id FK
    }
    expense_categories {
        uuid id PK
        uuid company_id "logical → identity"
        string name
    }
    budgets {
        uuid id PK
        uuid company_id "logical → identity"
        uuid category_id FK
    }
```

`invoices.counterparty_snapshot` е ключовата denormalization: legal document не трябва да
се променя, когато registry row на counterparty бъде редактиран по-късно.

---

## 5. Connections between databases

Тъй като няма cross-database FKs, всяка inter-database "connection" е **logical reference**,
resolve-ван runtime чрез един от три механизма:

| Mechanism | When used | Consistency | Example |
|---|---|---|---|
| **Sync HTTP** (service token) | Трябват fresh data сега | Strong (read-through) | business → registry counterparty lookup; business → document price reads |
| **Event (Redis Streams)** | Реакция на fact; decouple producers | Eventual, at-least-once | `tenant.created` → registry seeds + billing bonus; `token.usage` → billing ledger |
| **Denormalized snapshot** | Value трябва да е frozen / hot path | Point-in-time copy | invoice counterparty snapshot; `X-Roles` claims copied into the JWT |

### 5.1 Logical reference map

Solid = synchronous HTTP read; dashed = event-driven write/seed; всяка `company_id` /
`user_id` column е implicit reference към `identity` (начертана като central hub).

```mermaid
flowchart TB
    id[("identity")]
    agent[("agent")]
    mg[("modelgw")]
    kn[("knowledge")]
    reg[("registry")]
    doc[("document")]
    bill[("billing")]
    intg[("integration")]
    plat[("platform")]
    biz[("business")]

    %% identity is the universal tenant/user anchor
    agent -. "company_id / user_id" .-> id
    mg -. "company_id (kill switch)" .-> id
    kn -. "company_id / user_id" .-> id
    reg -. "company_id / user_id" .-> id
    doc -. "company_id / user_id" .-> id
    bill -. "company_id / user_id" .-> id
    intg -. "company_id / user_id" .-> id
    plat -. "company_id / user_id" .-> id
    biz -. "company_id / user_id" .-> id

    %% synchronous HTTP reads (resolve foreign data live)
    biz -->|"counterparty lookup<br/>(canonical role)"| reg
    biz -->|"price reads · PDF render"| doc
    doc -->|"row data for documents"| reg
    kn -->|"webdav/drive connection IO"| intg

    %% event-driven links (facts, seeds, metering)
    id -. "tenant.created → seed registries" .-> reg
    id -. "tenant.created → welcome bonus" .-> bill
    mg -. "token.usage → meter" .-> bill
    kn -. "document.ingested → cache bust" .-> agent

    %% audit + notifications sink (from every service)
    agent -. "audit.event" .-> plat
    biz -. "invoice.issued · stock.low" .-> plat
    bill -. "notification.requested" .-> plat
```

> Gateway няма база; той inject-ва verified `X-User-Id` / `X-Company-Id` /
> `X-Roles` headers, които позволяват на всяка услуга да *използва* `identity` references,
> без да вика `identity` по hot path-а.

### 5.2 Notable cross-database links explained

| From (DB.column) | To (DB.table) | How resolved | Notes |
|---|---|---|---|
| *every* `*.company_id` | `identity.companies` | header/JWT claim (no call) | tenant scoping; RLS enforced locally |
| *every* `*.user_id` | `identity.users` | header/JWT claim (no call) | row ownership |
| `business.invoices.counterparty_id` | `registry.registry_rows` | sync HTTP + **snapshot** | snapshot frozen at issue for legal immutability |
| `business.expenses.counterparty_id` | `registry.registry_rows` | sync HTTP | supplier linkage |
| `business.invoice_lines.unit_price` | `document.price_items` | sync HTTP | prices live in document-service |
| `knowledge.sync_folders.connection_ref` | `integration.connections` | sync HTTP | sync engine uses integration's file IO |
| `billing.usage_ledger.{feature,agent_id}` | model-gateway call metadata | **event** `token.usage` | attribution carried in event payload |
| `registry` system registries | `identity.companies` | **event** `tenant.created` | seeded on tenant creation |
| `billing.balances` welcome bonus | `identity.companies` | **event** `tenant.created` | initial token grant |
| `agent` context cache | `knowledge` | **event** `document.ingested` | invalidate stale retrieval cache |
| `platform.audit_log` | all services | **event** `audit.event` | central audit sink |
| `platform.notifications` | all services | **event** `notification.requested` | central delivery |

---

## 6. Shared infrastructure (не per-service databases)

### 6.1 Redis 7

Един logical Redis, използван от много услуги, но с **non-overlapping key namespaces** — това
е infrastructure, не shared database.

| User | Какво пази в Redis |
|---|---|
| gateway | rate-limit buckets, JWKS cache |
| identity | token-cleanup queue |
| agent | event streams, retention queues |
| model-gateway | balance-check queue, `token.usage` outbox flush |
| knowledge | embed/sync queues, **active project per user** |
| registry | event consumption |
| business | sweep queues |
| document | import queue |
| billing | top-up/rollup queues |
| integration | health-check queue |
| platform | email send queue |
| *all* | **event bus** (Redis Streams topics + consumer groups) |

Durability: AOF persistence (+ replica в production); billing-critical events също използват
**transactional outbox** в producing service, така че Redis outage ги забавя, но никога не ги
губи (виж [01 §7](./01-architecture-overview.md#7-asynchronous-work-and-events)).

### 6.2 S3 / MinIO

Binary blobs, които никога не принадлежат в Postgres:

| Owner | Objects | DB pointer |
|---|---|---|
| knowledge-service | uploaded document originals | `doc_files.s3_key` |
| agent-service | chat attachments | `attachments.s3_key` |
| document-service | generated PDFs/XLSX/DOCX artifacts | returned as signed URLs |

---

## 7. Вижте също

- [01 §6 — Identity, tenancy, authorization](./01-architecture-overview.md#6-identity-tenancy-and-authorization)
- [01 §7 — Asynchronous work and events](./01-architecture-overview.md#7-asynchronous-work-and-events)
- [02 — Service catalog](./02-service-catalog.md) (per-service *Data owned*)
- [06 — Architectural patterns](./06-architectural-patterns.md) (database-per-service, events, outbox)
- [07 §4 — Infrastructure dependency graph](./07-dependency-graphs.md#4-infrastructure-dependency-graph)
- Per-service detail: [services/](./services/README.md)
