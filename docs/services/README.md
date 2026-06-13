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
| [model-gateway](./model-gateway/README.md) | 8030 | The only LLM provider client — uniform API, metering events, kill switch |
| [knowledge-service](./knowledge-service/README.md) | 8040 | Library, parsing, embeddings, namespaced vector search, file sync |
| [registry-service](./registry-service/README.md) | 8050 | Dynamic registries, templates, canonical roles, clients, dashboard |
| [document-service](./document-service/README.md) | 8060 | Document templates, PDF/Excel generation, price list, margins, KSS |
| [billing-service](./billing-service/README.md) | 8070 | Token economy, Stripe, auto-top-up, limits and alerts |
| [integration-service](./integration-service/README.md) | 8080 | Google, IMAP/SMTP, WebDAV adapters + catalog + credentials vault |
| [platform-service](./platform-service/README.md) | 8090 | Notifications, transactional email, ops alerts, support tickets, audit sink, settings |
| [business-service](./business-service/README.md) | 8100 | Invoicing, inventory, spendings — typed ERP domains with hard invariants (post-parity) |

Boundaries that are deliberately **modules, not services**: conversation history (inside
agent-service) and notifications/support/audit/settings (one platform-service) — see
[02 § Deliberately merged](../02-service-catalog.md#deliberately-merged-boundaries).

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

## Shared conventions

Every service follows the standard layout, layering, and cross-cutting standards defined
in [02 — Service Catalog](../02-service-catalog.md) (hexagonal structure, `deps.py`
composition root, health/ready endpoints, structured logs, Alembic, tenant scoping + RLS),
and installs the [`x7-common` shared kernel](../libs/common/README.md) for config, auth
plumbing, the event bus, and observability.
