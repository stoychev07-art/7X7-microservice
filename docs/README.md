# 7x7 Microservice Platform — Architecture Documentation

Target architecture for the next generation of the 7x7 platform: an **agentic ERP system**
built as FastAPI microservices behind a single gateway, with LangGraph-powered agents and a
Next.js web frontend (more frontends — mobile app, Telegram/Viber bots — pluggable later).

This documentation set describes the *to-be* system. It covers the functionality of the
existing monolith at `7x7-platform`, minus features that the monolith's own audits flagged
as dead or redundant.

## Reading order

| # | Document | What it answers |
|---|----------|-----------------|
| 1 | [01-architecture-overview.md](./01-architecture-overview.md) | What are we building? Topology, principles, tech stack, cross-cutting concerns (auth, events, observability) |
| 2 | [02-service-catalog.md](./02-service-catalog.md) | What services exist? Responsibilities, API surfaces, data ownership, background jobs |
| 3 | [03-agent-platform.md](./03-agent-platform.md) | How do LangGraph agents and tools work, and how do we add new ones without touching core code? |
| 4 | [04-functional-coverage.md](./04-functional-coverage.md) | Where does each feature of the old monolith live in the new system — and what was deliberately dropped? |
| 5 | [05-migration-pros-and-cons.md](./05-migration-pros-and-cons.md) | Should we migrate? Honest pros, cons, risks, and a phased migration strategy |
| 6 | [06-architectural-patterns.md](./06-architectural-patterns.md) | Reference: every architectural pattern used (gateway, hexagonal, events, strangler, ReAct, …) — what it is, why it's here, what it costs |
| 7 | [07-dependency-graphs.md](./07-dependency-graphs.md) | Every dependency, drawn: whole-system graphs (full, sync HTTP, events, infrastructure) + one graph per service with internal structure and external dependencies |
| 8 | [08-database-architecture.md](./08-database-architecture.md) | Where does data live? Database-per-service catalog, per-DB schemas (ERDs), and how databases reference each other without cross-DB joins |
| 9 | [services/](./services/README.md) | Per-service starting points — one folder per service with responsibilities, API sketch, dependencies, and an implementation checklist |
| 10 | [libs/common/](./libs/common/README.md) | The `x7-common` shared kernel — config, auth plumbing, pooled HTTP, event bus, observability, cross-service DTOs |

## One-paragraph summary

All traffic — web, future mobile, future chat bots — enters through a single **API Gateway**
(FastAPI). Behind it sit small, independently deployable FastAPI services, each owning its
own database (database-per-service, no shared tables). The **Agent Service** hosts LangGraph
graphs discovered from per-agent folders (`manifest.yaml` + `graph.py`): adding an agent or a
tool never requires editing core runtime code. LLM calls from every service go through a
**Model Gateway** that abstracts providers (Anthropic, OpenAI, …) and emits token-usage
events consumed by the **Billing Service**. The frontend is a Next.js app that talks only to
the gateway; chat-channel adapters (Telegram, Viber) are thin clients of the same gateway API.

## Conventions used in these docs

- **Old / monolith** = `7x7-platform` (Node.js/Express, PM2, single Postgres).
- **New / target** = this repository (`7X7-microservice`).
- Service names are kebab-case (`agent-service`); internal Python packages are snake_case.
- Mermaid diagrams render in GitHub, Cursor, and most Markdown viewers.
