# External Services — Integrated Third-Party Engines

The platform is built from first-party microservices (see [../README.md](../README.md)), but
some hard, well-solved problems are **not** hand-rolled. Instead we run best-in-class
third-party engines and slot them **behind a port** owned by exactly one first-party service.
This folder documents those engines — services we make requests to — covering **what each is,
why we use it, what for, and how it is wired in**. Most are stateless request/response engines
(LiteLLM, Unstructured, Carbone); **Nango** and **Documenso** are the exceptions — each is
stateful (its own database) and adds an end-user surface (Nango's **Connect UI**, Documenso's
**signer pages**), so they are treated like small stateful services. **n8n** is a further
exception of a different kind: it is **not wrapped behind a single service's port** — it is a
platform-level **automation plane** that calls many services *through the gateway* (governed by
the gateway-only rule rather than by one owning adapter), and it is **platform/internal-only**,
never tenant-facing. **Node-RED** is the same kind of exception for the IoT vertical: a
gateway-only **ingestion edge** in front of [iot-service](../iot-service/README.md) that decodes
industrial protocols and posts normalized readings through the gateway. **Jaeger** is an
exception of yet another kind: it is an **observability backend** owned by *no* service —
**every** service pushes OpenTelemetry spans to it via the `x7-common` exporter — and it is an
internal engineering/ops console, never on the application request path.

> **Datastores live elsewhere.** Self-hosted data stores (PostgreSQL, Redis, MinIO, the
> **Qdrant** vector database, and **TimescaleDB** — the `iot` time-series DB, which is
> PostgreSQL + an extension) are infrastructure owned by their service, not "services we
> call" — they are documented in [08 — database architecture](../../08-database-architecture.md)
> and the [07 infrastructure dependency graph](../../07-dependency-graphs.md#4-infrastructure-dependency-graph).

## The rule: engines behind ports, owned by one service

Every external engine sits behind a port (hexagonal adapter) inside a single owning service.
Callers never talk to the engine directly — they call the owning service's stable API. This
keeps the engine an **implementation detail** that can be swapped, and keeps tenancy,
metering, auth, and business rules in our code.

| Engine | Kind | Owner service | Internal endpoint | Public? |
|---|---|---|---|---|
| [LiteLLM](./litellm/README.md) | LLM provider gateway | [model-gateway](../model-gateway/README.md) (8030) | `litellm:4000` | No — internal only |
| [Unstructured](./unstructured/README.md) | Document parsing / OCR | [knowledge-service](../knowledge-service/README.md) (8040) | `unstructured:8000` | No — internal only |
| [Carbone](./carbone/README.md) | Document rendering (PDF/DOCX/XLSX) | [document-service](../document-service/README.md) (8060) | `carbone:4000` | No — internal only |
| [Nango](./nango/README.md) | OAuth / token backend (outbound) | [integration-service](../integration-service/README.md) (8080) | `nango:3003` (proxy), `nango:3009` (Connect UI) | No — internal only; admin UI via mesh VPN |
| [Documenso](./documenso/README.md) | E-signature (stateful) | [integration-service](../integration-service/README.md) (8080) | `documenso:3000` | No — internal only; signer pages via gateway, admin via mesh VPN |
| [n8n](./n8n/README.md) | Automation plane (stateful, **platform-only**) | *none* — gateway-only, calls many services (not port-wrapped) | `n8n:5678` | No — internal only; editor UI behind Authentik SSO (staff), never public |
| [Node-RED](./node-red/README.md) | IoT ingestion edge (stateful, decodes MQTT/Modbus/OPC UA) | *none* — gateway-only, posts to [iot-service](../iot-service/README.md) (not port-wrapped) | `node-red:1880` | No — internal only; editor UI behind Authentik SSO (staff), never public |
| [Jaeger](./jaeger/README.md) | Distributed-tracing backend + UI (observability) | *none* — fed by **all** services via the `x7-common` OTel exporter (not port-wrapped) | `jaeger:4317`/`:4318` (OTLP ingest), `jaeger:16686` (UI) | No — internal only; UI behind Authentik SSO (staff)/SSH tunnel, never public |

> **Single public door preserved.** None of these engines is exposed on the public internet for
> *application* traffic — only the API gateway (`:8000`) is. Each engine is reachable only on the
> internal network by its owning service's adapter (n8n and **Node-RED** are the exceptions:
> they call services / post readings through the gateway, never port-wrapped). Admin/ops
> surfaces (where they exist) are never an open public ingress — they are reached either over a
> private **mesh VPN** (Nango/Documenso admin) or behind **Authentik SSO** forward-auth for
> staff-only consoles (n8n editor, **Node-RED editor**, **Jaeger UI** — or an SSH tunnel for
> quick looks), consistent with Authentik fronting the internal ops consoles
> ([09 §3.1](../../09-industry4z-platform-integration.md#31-identity--authentik-)).

## Why this approach

- **Don't build what's already solved.** Provider routing, document parsing, and OAuth/token
  management are deep problems with mature engines. We spend our effort on the business layer
  those engines do not model.
- **Swappable by design.** Because each engine is behind a port, replacing it is "write one
  adapter," not "rewrite the service." This is the same pattern used to adopt them in the
  first place (see [09 — Industry4Z platform integration](../../09-industry4z-platform-integration.md)).
- **Boundaries stay ours.** Tenancy, the `token.usage` billing truth, per-tenant kill switch,
  namespace scoping, and access control all live in first-party code — the engine never owns
  a product concern.

## References

- [09 — Industry4Z platform integration](../../09-industry4z-platform-integration.md) — the
  analysis that selected these engines and the architecture changes to adopt them.
- [07 — Dependency graphs](../../07-dependency-graphs.md) — where each engine appears as a
  dependency of its owning service.
- [01 — Architecture overview](../../01-architecture-overview.md#4-technology-stack) — the
  technology-stack table.
