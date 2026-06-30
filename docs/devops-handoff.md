# DevOps Handoff — Platform Infrastructure & Deployment

A single-page brief for the DevOps team: what we run, the datastores, how containers are
built and deployed, and recommended server sizing. For full detail see
[11 — Deployment & Operations](./11-deployment.md), [10 — Phase-1 Co-Deployment](./10-phase1-co-deployment.md),
and [08 — Database Architecture](./08-database-architecture.md).

---

## 1. TL;DR

- **One Git repo → one Docker image.** Every service is the same image; a `SERVICES` env var
  decides which services a given container runs.
- **13 logical services**, run in production as **7 application deployables** (each with a
  `web` + a `worker` container) + the **gateway** as the only public entry point.
- **Database-per-service**: 12 logical Postgres databases on **one PostgreSQL 16 cluster**
  (one scoped role per DB, no cross-schema joins) + a **separate TimescaleDB cluster** for IoT.
- Shared backing stores: **Redis 7**, **Qdrant** (vectors), **S3 / MinIO** (files),
  **TimescaleDB** (IoT time-series).
- Plus a handful of **self-hosted third-party engines** (LiteLLM, Carbone, Unstructured,
  Nango, Documenso, Authentik, Node-RED) on the internal network.
- Phase-1 target: **a single adequately-sized host** (Docker Compose). Kubernetes is *not*
  required for correctness — scale out later by adding replicas.

---

## 2. Services (short descriptions)

All services are FastAPI (Python 3.12) apps. Ports below are the in-container dev ports;
in production they're reached over the internal network.

| Service | Port | What it does |
|---|---|---|
| **gateway** | 8000 | The single public entry point. Reverse proxy, JWT/JWKS validation, rate limiting, CORS, webhook passthrough. **No database.** |
| **identity-service** | 8010 | Users, tenants (companies), login (Authentik OIDC + local), JWT issuance, RBAC roles/permissions. |
| **agent-service** | 8020 | The LangGraph AI agent runtime + conversation history (sessions/messages). The streaming hot path. |
| **model-gateway** | 8030 | The only service that talks to LLM providers (via LiteLLM). Token metering, kill switch. |
| **knowledge-service** | 8040 | Document library, parsing/OCR, embeddings + vector search (RAG), WebDAV/Drive sync. |
| **registry-service** | 8050 | Dynamic user-defined tables (CRM, contracts, tasks, etc.), templates, audit. |
| **business-service** | 8100 | Typed ERP domain: invoicing, inventory/stock ledger, expenses. |
| **document-service** | 8060 | Document templates + rendering (PDF/DOCX/XLSX via Carbone), price lists, margins, KSS. |
| **billing-service** | 8070 | Token balances, usage metering, Stripe checkout + webhooks, packages. |
| **integration-service** | 8080 | External connectivity: Google/Gmail, IMAP/SMTP, WebDAV, e-signature (Documenso), OAuth (Nango). |
| **platform-service** | 8090 | Notifications (email/Telegram/in-app), support tickets, central audit sink, settings. |
| **iot-service** | 8110 | IoT device/sensor registry, time-series storage (TimescaleDB), continuous aggregates, per-tenant alerting. Readings arrive normalized from the Node-RED ingestion edge. |
| **manufacturing-service** | 8120 | Production orders, BOM/routing, MRP, capacity planning, MES terminals + SCADA↔ERP exchange. |

---

## 3. Infrastructure & datastores

### 3.1 Databases (PostgreSQL 16, one cluster)

Database-per-service: **12 logical databases on one PostgreSQL 16 cluster** (each with its own
login role that can only access its own database — no cross-schema joins, enforced by grants) +
**one separate TimescaleDB cluster** for the `iot` time-series database.

| Database | Owning service | Notable |
|---|---|---|
| `identity` | identity-service | RS256 keypair for JWKS |
| `agent` | agent-service | LangGraph checkpoint tables |
| `modelgw` | model-gateway | AES-GCM encrypted provider credentials |
| `knowledge` | knowledge-service | pairs with **Qdrant** for vectors |
| `registry` | registry-service | JSONB rows, Row-Level Security |
| `document` | document-service | JSONB templates |
| `billing` | billing-service | append-only usage ledger |
| `integration` | integration-service | AES-GCM credential vault (non-OAuth) |
| `platform` | platform-service | notifications, audit, support |
| `business` | business-service | per-tenant invoice sequences |
| `iot` | iot-service | **separate TimescaleDB cluster** (Postgres 16 + Timescale extension): hypertables, continuous aggregates, retention/compression |
| `manufacturing` | manufacturing-service | per-tenant order sequences, RLS |

> `iot` runs on its own **TimescaleDB** cluster (Postgres 16 + the Timescale extension). The
> other 12 are logical DBs on the shared cluster — database-per-service still holds.

### 3.2 Shared backing stores

| Component | Image | Used for |
|---|---|---|
| **Redis 7** | `redis:7` (AOF on) | Cache, rate limits, `arq` job queues, event bus (Redis Streams). Run with AOF + a replica in prod. |
| **Qdrant** | `qdrant/qdrant` | Vector store for knowledge-service RAG. |
| **S3 / MinIO** | `minio/minio` (dev) / any S3 in prod | Document originals, generated PDFs/XLSX/DOCX, chat attachments. Bucket-per-owning-service. |

### 3.3 Self-hosted third-party engines (internal network only)

Each sits behind a port owned by one service. None is publicly exposed for application traffic.

| Engine | Image | Owner | Endpoint | Stateful? |
|---|---|---|---|---|
| **LiteLLM** | `ghcr.io/berriai/litellm` | model-gateway | `litellm:4000` | No |
| **Carbone** | built (`infra/carbone`) | document-service | `carbone:4000` | No |
| **Unstructured** | `unstructured-io/unstructured-api` | knowledge-service | `unstructured:8000` | No |
| **Nango** | `nangohq/nango-server` | integration-service | `nango:3003` / `:3009` | **Yes** (own pg/redis) |
| **Documenso** | `documenso/documenso` | integration-service | `documenso:3000` | **Yes** (own pg) |
| **Authentik** | self-hosted | identity (OIDC IdP) | internal | **Yes** (own pg) |
| **Node-RED** | `nodered/node-red` | iot-service (ingestion edge) | `node-red:1880` | **Yes** (own volume) |

> Node-RED is the IoT ingestion edge: it decodes industrial protocols (MQTT/Modbus/OPC UA) and
> posts normalized readings to iot-service **through the gateway** (not port-wrapped). Its editor
> UI is staff-only, behind Authentik SSO — never public.
>
> Optional: **Ollama** (local LLM, behind LiteLLM) for cost/privacy, and **n8n** (internal
> automation plane). Stand these up only when needed.

---

## 4. How containers are built & deployed

### 4.1 One image, many roles

- All services build from a **single `Dockerfile`** at the repo root (`python:3.12-slim`, `uv`
  installs the whole workspace). CI builds and caches this image **once**, tagged per commit.
- A container's behaviour is set entirely by the **`SERVICES` env var**, e.g.
  `SERVICES=agent,model-gateway,knowledge`. The entrypoint mounts only those routers; the worker
  entrypoint registers only those background jobs.
- **`web` and `worker` are separate containers** off the same image, per deployable — so a heavy
  background job (embedding sweep, nightly MRP) never starves HTTP latency.

### 4.2 Production deployable map (7 + gateway)

| Deployable | `SERVICES` value | Worker? | Public? |
|---|---|---|---|
| **gateway** | `gateway` | no | **Yes — the only one** |
| **identity** | `identity` | yes | no |
| **ai-plane** | `agent,model-gateway,knowledge` | yes | no |
| **erp-core** | `registry,business,document` | yes | no |
| **ops-plane** | `billing,integration,platform` | yes | no |
| **iot** | `iot` | yes | no |
| **manufacturing** | `manufacturing` | yes | no |

> `iot` and `manufacturing` run **split from day one** (their own deployables) — their load
> profiles (time-series ingest, nightly MRP runs) have nothing to do with the core scaling curve,
> and `iot` needs the separate TimescaleDB cluster.

Grouping is by **scaling profile + trust boundary + statefulness**, not domain. Splitting any
group into its own container later is a **config change** (new container, point its
`DATABASE_URL`, flip a `deps.py` adapter, update the gateway routing table) — never a rewrite.

### 4.3 Deploy flow

1. **Build** the single image, tag per commit.
2. **Migrate**: a `migrate` job (same image) runs `alembic upgrade head` for each service in its
   deployable **before** new code serves traffic. Migrations are backward-compatible
   (expand/contract) so rolling deploys are zero-downtime.
3. **Roll** the `web` / `worker` containers. The gateway gates traffic on `/ready`.
4. Networking: **TLS terminates at the gateway** (or a reverse proxy in front). Everything else
   is internal-only. Webhooks (Stripe, Brevo, Documenso, Nango) pass through the gateway with raw
   body preserved.

### 4.4 Config & secrets

- 12-factor: **all config via environment variables**. Dev uses a git-ignored `.env`; prod
  injects from a **secret manager** at deploy time.
- **Secrets never in code or images.** One owner per external credential (LLM keys → model-gateway,
  Stripe → billing, Nango/Documenso → integration, Brevo/Telegram → platform).
- Each service connects with its **own DB role** (its own `DATABASE_URL`).

### 4.5 Health & observability

- `GET /health` (liveness) and `GET /ready` (DB + Redis reachable — orchestrator gates on this).
- **Structured JSON logs** (structlog) with `request_id`, `user_id`, `company_id`, `service`.
- **OpenTelemetry** traces (gateway starts the trace, every hop continues it) + **Sentry** for errors.

---

## 5. Server recommendations

> These are starting recommendations for a single-host phase-1 deployment serving SME tenants.
> Tune from real load — the architecture scales by adding replicas, not by re-architecting.

### 5.1 Phase-1 — single application host

Runs the gateway + 7 app deployables (web + worker each) and the stateless backing engines
(LiteLLM, Carbone, Unstructured, Node-RED). Cloud LLMs (Anthropic) mean no GPU is required.

| Resource | Recommendation |
|---|---|
| **vCPU** | 8–12 vCPU (minimum 4) — the iot ingest + nightly MRP workers add load on top of the core |
| **RAM** | 24–32 GB (the LLM/RAG hot path and document rendering are the memory drivers) |
| **Disk** | 100+ GB SSD for the host; object storage separate (see below) |
| **OS** | Ubuntu LTS (or any modern Linux with Docker Engine + Compose v2) |

### 5.2 Data plane (recommend a separate host / managed instances)

| Component | Recommendation |
|---|---|
| **PostgreSQL 16** | Managed instance strongly preferred. Start 4 vCPU / 16 GB RAM / 100 GB SSD with room to grow; enable automated backups + PITR. |
| **Redis 7** | 1–2 vCPU / 4 GB RAM, AOF persistence on, + a replica in prod. |
| **Qdrant** | 2 vCPU / 4–8 GB RAM, SSD-backed volume (grows with document/vector count). |
| **S3 / MinIO** | S3 in prod (no sizing needed) or a MinIO host with ample disk + versioning enabled. |
| **TimescaleDB** (IoT) | Separate cluster (Postgres 16 + Timescale extension). Start 4 vCPU / 16 GB RAM / SSD; size disk for time-series ingest volume + retention window. Compression/retention policies keep raw-row growth in check. |

### 5.3 Scale-out path (in order, when load shows)

1. **Gateway replicas** behind a load balancer (stateless — trivial).
2. **ai-plane replicas** — the chat/LLM hot path; scale web and worker independently.
3. **Split the hottest member out** (model-gateway or knowledge) when contention appears.
4. **Split erp-core / ops-plane members** as individual load or team ownership demands.

Each step is a config/topology change (replica count, `SERVICES` value, routing table) on the
**same image**.

### 5.4 GPU note

Not required for the default setup (LLMs are cloud Anthropic via LiteLLM). A GPU host is only
needed if you self-host local models (Ollama) for cost/privacy — separate that onto its own
GPU-equipped node.

---

## 6. Backups & durability

| Asset | Mechanism |
|---|---|
| PostgreSQL (all DBs) | Scheduled `pg_dump` / PITR; back up per-DB so a restore can be service-scoped. |
| Redis | AOF + a replica; queued events/jobs survive restart. |
| Object storage | S3/MinIO versioning + lifecycle policies. |
| Critical billing events | Transactional outbox in the producer — losing an event requires losing the DB, not just Redis. |
| External backers | Nango (own pg/redis), Documenso (own pg), Authentik (own pg) — back these up too. |

---

## 7. Quick reference (dev)

| Task | Command |
|---|---|
| Boot everything | `docker compose up --build` |
| Run migrations | `docker compose run --rm migrate` |
| Follow one deployable | `docker compose logs -f ai-plane` |
| Scale the gateway | `docker compose up -d --scale gateway=3` |

See [10 §7](./10-phase1-co-deployment.md#7-sample-dev-docker-compose-sketch) for the reference
`docker-compose.yml`.
