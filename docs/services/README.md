# Services — Per-Service Documentation

One folder per service. Each README is the starting point for implementing that service:
responsibilities, API sketch, owned data, dependencies, design notes, and an
implementation checklist. As services get built, these folders grow (API contracts,
runbooks, ADRs).

| Service | Port (dev) | One-liner |
|---|---|---|
| [gateway](./gateway/README.md) | 8000 | Single public entry point — routing, JWT verification, rate limiting, SSE passthrough |
| [identity-service](./identity-service/README.md) | 8010 | Users, companies, roles, JWT issuance, OAuth, JWKS, service tokens |
| [agent-service](./agent-service/README.md) | 8020 | LangGraph agent runtime — folder-discovered agents, tool catalog, approval interrupts, conversation store |
| [model-gateway](./model-gateway/README.md) | 8030 | The only LLM provider client — thin owner backed by **LiteLLM** (multi-provider: Claude + local Ollama); uniform API, metering events, kill switch |
| [knowledge-service](./knowledge-service/README.md) | 8040 | Library, parsing, embeddings, namespaced vector search, file sync |
| [registry-service](./registry-service/README.md) | 8050 | Dynamic registries, templates, canonical roles, clients, dashboard |
| [document-service](./document-service/README.md) | 8060 | Document templates, PDF/Excel/Word generation via **Carbone**, price list, margins, KSS |
| [billing-service](./billing-service/README.md) | 8070 | Token economy, Stripe, auto-top-up, limits and alerts |
| [integration-service](./integration-service/README.md) | 8080 | Google, IMAP/SMTP, WebDAV, **e-signature** adapters + catalog; OAuth via **Nango**, e-sign via **Documenso**, non-OAuth secrets in local vault |
| [platform-service](./platform-service/README.md) | 8090 | Notifications, transactional email, ops alerts, support tickets, audit sink, settings |
| [business-service](./business-service/README.md) | 8100 | Invoicing, inventory, spendings — typed ERP domains with hard invariants (post-parity) |
| [iot-service](./iot-service/README.md) | 8110 | IoT vertical — device registry, **TimescaleDB** time-series, ingestion from **Node-RED**, own multi-tenant alerting (no Grafana) |
| [manufacturing-service](./manufacturing-service/README.md) | 8120 | Manufacturing vertical — production orders, BOM/routing, **MRP**, capacity, **MES terminals**, order-linked SCADA↔ERP (net-new) |

Boundaries that are deliberately **modules, not services**: conversation history (inside
agent-service) and notifications/support/audit/settings (one platform-service) — see
[02 § Deliberately merged](../02-service-catalog.md#deliberately-merged-boundaries).

## External engines (third-party, behind ports)

Some problems are not hand-rolled — we run best-in-class third-party engines **behind a port**
owned by one first-party service. They are internal-only (never publicly exposed) and
documented in [external/](./external/README.md):

| Engine | Owner service | Internal endpoint | Used for |
|---|---|---|---|
| [LiteLLM](./external/litellm/README.md) | model-gateway (8030) | `litellm:4000` | Multi-provider LLM gateway (routing, fallback, budgets) |
| [Unstructured](./external/unstructured/README.md) | knowledge-service (8040) | `unstructured:8000` | Document parsing / OCR at ingest |
| [Carbone](./external/carbone/README.md) | document-service (8060) | `carbone:4000` | Template → PDF/DOCX/XLSX rendering (stateless) |
| [Nango](./external/nango/README.md) | integration-service (8080) | `nango:3003` · `nango:3009` | OAuth/token backend for outbound SaaS (Google, …); token refresh + Connect UI |
| [Documenso](./external/documenso/README.md) | integration-service (8080) | `documenso:3000` | E-signature on generated documents (stateful — own Postgres) |
| [Node-RED](./external/node-red/README.md) | iot-service (8110) — gateway-only edge | `node-red:1880` | IoT protocol ingestion (MQTT/Modbus/OPC UA → normalized readings via gateway) |

Self-hosted **data stores** (PostgreSQL, Redis, MinIO, the **Qdrant** vector database, and
**TimescaleDB** — the `iot` time-series DB, Postgres + extension) are infrastructure owned by
their service — see [08 — database architecture](../08-database-architecture.md)
and the [07 infrastructure dependency graph](../07-dependency-graphs.md#4-infrastructure-dependency-graph).

Each service's full **dependency graph** (internal layering + callers, callees, events,
infrastructure, third-party systems) is drawn in
[07 — Dependency Graphs](../07-dependency-graphs.md), alongside the whole-system views.

Deferred / future (no folder until decided): **schematics-service** (electrical-schematic
extraction vertical — see [04 §5](../04-functional-coverage.md)), **telegram-adapter** /
**viber-adapter** (thin channel clients of the gateway chat API).

## Build order

Follow the strangler phases in [05 §5](../05-migration-pros-and-cons.md):
gateway → identity + model-gateway → agent + knowledge →
registry + document → billing + integration + platform → business-service
(after parity).

Net-new verticals (no monolith source) build on their own track once their dependencies
exist: **iot-service** (after the gateway + identity) and **manufacturing-service** (after
business-service, since it reserves/consumes stock there) — see
[04 §7](../04-functional-coverage.md#7-new-manufacturing-vertical-scada--mes--mrp).

## Shared conventions

Every service follows the standard layout, layering, and cross-cutting standards defined
in [02 — Service Catalog](../02-service-catalog.md) (hexagonal structure, `deps.py`
composition root, health/ready endpoints, structured logs, Alembic, tenant scoping + RLS),
and installs the [`x7-common` shared kernel](../libs/common/README.md) for config, auth
plumbing, the event bus, and observability.
