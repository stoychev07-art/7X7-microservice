# 06 — Architectural Patterns Explained

A reference for every named pattern the architecture relies on: what the pattern is, where
it shows up in this system, why it was chosen over the alternatives, and what it costs.
Read this if a term in the other documents ("port", "strangler", "thin event", …) is
unfamiliar or you want the reasoning behind a design rule.

---

## 1. Structural patterns (system shape)

### 1.1 Microservices with bounded contexts (DDD)

**What it is.** The system is decomposed into independently deployable services, each
aligned with a *bounded context* from Domain-Driven Design — a business area with its own
vocabulary, model, and invariants (identity, billing, knowledge, registries, …).

**Where.** The 10 services (11 with the post-parity business-service) in
[02-service-catalog.md](./02-service-catalog.md). The boundaries were not invented from
scratch: they follow the highest-cohesion module clusters already visible in the monolith
(`core/auth`, `core/registries`, `core/payments`, …), which is the safest way to draw
service lines — along seams the domain has already proven. The inverse rule also applied:
boundaries that don't pay for their own deployable stay modules (conversation history in
agent-service; notifications/support/audit/settings in one platform-service — see
[02 § Deliberately merged](./02-service-catalog.md#deliberately-merged-boundaries)).

**Why.** Independent scaling of hot paths (chat vs. CRUD vs. PDF rendering), failure
isolation, and the ability to evolve the agent platform aggressively without destabilizing
billing or auth.

**Trade-off.** Distributed-systems complexity: network failures between services, more
deployment units, cross-service debugging. This is why the architecture pairs the pattern
with two mitigations: a strict call-matrix (only agent-service fans out) and the rule
*"boundaries are code boundaries first, deployment boundaries second"* — services may be
co-deployed in one container until load forces a split.

### 1.2 API Gateway

**What it is.** A single edge service through which all external traffic enters. It owns
cross-cutting concerns (authentication, rate limiting, routing, streaming passthrough) and
contains zero business logic.

**Where.** The `gateway` service ([01 §5](./01-architecture-overview.md#5-the-api-gateway)).

**Why.** Three reasons specific to this system: (a) multiple current and future frontends
(web, mobile, Telegram/Viber) must hit identical APIs with identical security; (b) the
monolith's lesson — its route-mount *ordering* in `server.js` was load-bearing and its rate
limiter was duplicated 14 times — argues for exactly one place where edge policy lives;
(c) during migration the gateway's routing table *is* the strangler mechanism (§4.1).

**Trade-off.** The gateway is a single point of failure and a potential bottleneck — it
must stay thin (no aggregation, no transformation) and be the first thing replicated.

### 1.3 Database-per-Service

**What it is.** Each service owns its data store exclusively. No other service may connect
to it; cross-service data moves over HTTP APIs or events.

**Where.** Every DB-owning service has its own logical PostgreSQL database
(`identity`, `registry`, `knowledge`, …). The vector store (pgvector) belongs to
knowledge-service alone; agents query it only through the `/search` API.

**Why.** The monolith's single schema with ~80 `core.*` tables let any module join any
table — invisible coupling that made every change a platform-wide risk. Private databases
make every dependency an explicit, versionable contract, and let each service pick the
right storage shape (JSONB rows for registries, vectors for knowledge).

**Trade-off.** No cross-service joins and no cross-service transactions. Aggregations move
into the owning service (dashboard briefing lives in registry-service); consistency across
services becomes *eventual* and is handled with idempotent event consumers (§3.2).

### 1.4 Multi-tenancy with defense-in-depth (RLS)

**What it is.** All tenants share the same services and databases; every row carries a
tenant ID, every query is tenant-scoped, and PostgreSQL Row-Level Security enforces the
scope a second time at the database layer.

**Where.** Tenant context (`X-Company-Id`) is verified at the gateway, propagated in
headers, and applied in every service. RLS policies mirror the monolith's existing GUC
approach (`app.current_tenant`).

**Why.** Tenant isolation bugs are the most damaging class of bug in a B2B platform. Two
independent layers (application scoping + RLS) mean a forgotten `WHERE company_id = …`
becomes a query returning nothing, not a data leak.

**Trade-off.** RLS adds friction to migrations and debugging (sessions must set the GUC),
and slightly complicates admin cross-tenant reads (explicit bypass role, audited).

---

## 2. Patterns inside each service

### 2.1 Hexagonal architecture (Ports & Adapters)

**What it is.** The domain core depends only on **ports** — interfaces (Python
`Protocol`s) describing what the domain needs (`LLM`, `Retriever`, `ConversationStore`,
`VectorStore`). **Adapters** implement those ports and are the only modules allowed to
import a driver, SDK, or know a URL. Dependencies always point inward: edge → domain ←
adapters.

**Where.** The standard service layout in [02](./02-service-catalog.md) (`routes/ →
services/ → ports/ ← adapters/`) and the swap recipes in
[03 §5](./03-agent-platform.md#5-swapping-infrastructure).

**Why.** It makes the expensive things cheap: swapping the vector store, repointing a tool
adapter from the monolith to a new service during migration, or replacing an LLM provider
is *one new adapter + one wiring line* — the domain and every agent stay untouched. It also
makes domain logic testable with in-memory fakes instead of mocked HTTP.

**Trade-off.** More files and a level of indirection that feels like ceremony in tiny
services. The discipline pays off only if it's enforced — hence the rule that a driver
import outside `adapters/` fails code review.

### 2.2 Repository pattern + DTO/model separation

**What it is.** Database access is encapsulated in repository classes behind ports, and the
shapes that cross the API (`Pydantic` DTOs) are deliberately distinct from storage shapes.

**Where.** Every DB-owning service's `adapters/` layer; the rule "DTOs ≠ DB models" in
[03 §5](./03-agent-platform.md#case-b--swap-a-database-in-a-db-owning-service).

**Why.** The API contract can stay stable while storage evolves (add columns, change
indexes, even change engines). The monolith already used repositories (35 of them) — this
carries the habit over and formalizes the DTO boundary it lacked.

**Trade-off.** Mapping code. Kept tolerable by Pydantic's `model_validate` and by not
inventing DTOs where the storage shape genuinely is the contract.

### 2.3 Dependency injection with a composition root

**What it is.** Objects don't construct their own dependencies; everything is wired in one
place — `deps.py` — using FastAPI's dependency system. That file is the *composition root*:
the only place adapters are instantiated and bound to ports.

**Where.** Every service; all "swap infrastructure" recipes end with "rewire one line of
`deps.py`".

**Why.** Swappability (§2.1) is only real if there is exactly one wiring point. It also
gives tests a single seam to override (inject fakes) without monkeypatching.

**Trade-off.** Indirection — to find *which* concrete class serves a port you look at
`deps.py`, not the call site. That is the point, but it surprises newcomers.

---

## 3. Communication patterns

### 3.1 Synchronous request/response with context propagation

**What it is.** Service-to-service calls are plain HTTP (httpx, pooled, internal network)
carrying verified identity headers (`X-User-Id`, `X-Company-Id`, `X-Roles`,
`X-Request-Id`) plus a short-lived **service token** — a client-credentials style proof
that the caller is a platform service, not just something on the network.

**Where.** All arrows in the call matrix ([02](./02-service-catalog.md)); header injection
rules in [01 §5](./01-architecture-overview.md#5-the-api-gateway).

**Why.** Used for *queries and commands that need an answer now* (agent tools reading
registry rows). HTTP + OpenAPI keeps contracts explicit and the TS client generatable.

**Trade-off.** Latency per hop and temporal coupling (callee must be up). Mitigated by
keeping the call graph shallow (agent-service is the only fan-out hub) and by moving
everything that *doesn't* need an immediate answer to events (§3.2).

### 3.2 Event-driven pub/sub with thin events and idempotent consumers

**What it is.** Services publish *facts* (`tenant.created`, `token.usage`,
`document.ingested`) to Redis Streams topics; consumer groups process them independently.
Events are **thin** — IDs and minimal payload; consumers fetch details over HTTP if needed.
Every consumer is **idempotent** because delivery is at-least-once.

**Where.** [01 §7](./01-architecture-overview.md#7-asynchronous-work-and-events) — the
topic table and the fan-out diagram.

**Why.** Decouples producers from consumers: the model-gateway doesn't know billing exists;
registry-service seeds system registries on tenant creation without identity-service
knowing about registries. New consumers attach without touching producers — the same
open/closed property the agent platform has.

**Trade-off.** Eventual consistency (a token balance lags its LLM call by milliseconds to
seconds) and the operational need to monitor consumer lag and dead-letter handling. Where
an event *must not* be lost relative to a DB write, the producing service uses an outbox
table flushed by its worker (write the row and the pending event in one transaction).
`token.usage` is in that class — it is billing data, so the model-gateway always
publishes it through its outbox; Redis runs with AOF persistence + a replica in
production for everything else (see [01 §7](./01-architecture-overview.md#7-asynchronous-work-and-events)).

### 3.3 Job queue / private workers

**What it is.** Each service owns arq workers reading its own Redis queues for deferred or
heavy work (embedding, file sync, email sends, retention purges). Queues are private —
no service enqueues into another's.

**Where.** The `workers/` folder of each service; job lists per service in
[02](./02-service-catalog.md).

**Why.** Keeps request latency flat and isolates background load — the monolith learned
this the hard way when Drive sync starved its general worker, forcing the `sync-worker.js`
split. Private queues preserve the database-per-service principle for work, not just data.

**Trade-off.** Job state is another thing to observe; cross-service workflows must
communicate via events, never by reaching into a neighbor's queue.

---

## 4. Evolution & extensibility patterns

### 4.1 Strangler Fig (migration)

**What it is.** Instead of a big-bang rewrite, a facade (the gateway) sits in front of the
legacy system; new services take over route-by-route until the old system serves nothing
and is removed — the new system "strangles" the old one gradually.

**Where.** The entire migration plan in
[05 §5](./05-migration-pros-and-cons.md#5-risk-reducing-migration-strategy-strangler-not-big-bang):
the gateway proxies everything to the monolith on day one, then each phase flips path
prefixes to new services. Agent tools use adapter clients pointed at the monolith first,
new services later — a config flip thanks to ports & adapters (§2.1).

**Why.** Real tenants use the platform daily; a cutover that can be advanced (and rolled
back) per route and per tenant converts an existential risk into a sequence of small ones.

**Trade-off.** Double infrastructure and double bookkeeping for the duration, plus a
feature freeze on the monolith to keep parity a fixed target.

### 4.2 Plugin architecture: convention over configuration + registry discovery

**What it is.** Extending the system means *adding files that follow a convention*, not
editing core code. A registry discovers extensions at startup by scanning the filesystem
(reflection), validates their manifests, and mounts them; a broken extension is skipped,
never fatal.

**Where.** Three places, deliberately the same shape:
- **Agents** — `app/agents/<id>/manifest.yaml` + optional `graph.py`
  ([03 §1](./03-agent-platform.md#1-how-extensibility-is-achieved)).
- **Tools** — one module per tool + a single registry list
  ([03 §4](./03-agent-platform.md#4-tools-the-shared-catalog)).
- **Integration adapters** — folder + manifest + adapter class in integration-service.

**Why.** This is the system's core extensibility requirement: agents and tools must be
addable by a developer (or eventually, generated) without platform knowledge. The
open/closed principle made operational: open for extension, closed for modification.

**Trade-off.** Indirection at startup ("where did this route come from?") — mitigated by
logging every discovered/skipped extension — and the need to keep the manifest schema
stable, since it is a public contract.

### 4.3 Declarative manifests (configuration as contract)

**What it is.** An extension's *behaviour envelope* — model, allowed tools, RAG
namespaces, channels, prompts — is data, not code. The runtime reads the manifest and
wires/constrains the extension accordingly.

**Where.** Agent `manifest.yaml` ([03 §2](./03-agent-platform.md#2-anatomy-of-an-agent-folder));
integration adapter manifests; registry templates (a business-level instance of the same
idea — a registry is defined by declarative column specs with canonical roles, not code).

**Why.** Data is reviewable, diffable, validatable (Pydantic), and editable without
programming. Most behaviour changes (give an agent a tool, change its model, scope its
RAG) become one-line YAML edits.

**Trade-off.** Manifests can silently drift from reality if the runtime doesn't validate
them strictly — hence parse-into-Pydantic-or-skip at discovery time.

### 4.4 Template Method (the default agent graph)

**What it is.** A base implementation defines the skeleton of an algorithm; specializations
override only the steps that differ. Here: `build_default_graph()` provides the complete
load-context → ReAct loop → approval → answer pipeline, and `default_nodes()` exposes its
steps as reusable LangGraph nodes a custom `graph.py` can recompose.

**Where.** [03 §1 Pillar 2 and §2](./03-agent-platform.md).

**Why.** The marginal cost of a new agent is only its *unique* behaviour — a manifest-only
agent ships zero code; a custom agent writes one planning node, not a whole loop.

**Trade-off.** The default graph is a shared dependency: changes to it affect every
manifest-only agent, so it needs its own regression tests.

---

## 5. Agentic patterns

### 5.1 ReAct (reason + act tool loop)

**What it is.** The model alternates between reasoning and tool calls: it sees the tool
specs, requests a call, the runtime executes it, appends the result to the conversation,
and re-invokes the model — until the model answers without requesting tools.

**Where.** The `call_model ⇄ run_tools` cycle in the default graph
([03 §1](./03-agent-platform.md)); iteration-capped to prevent runaway loops.

**Why.** It is the simplest agent pattern that handles open-ended business questions
("draft an offer for client X") — and it matches what the monolith's hand-rolled loop
already did, minimizing behavioral migration risk.

**Trade-off.** Latency and token cost grow with each iteration; tools must be designed so
common workflows resolve in few calls (e.g. upfront context loading, per-turn dedupe).

### 5.2 Human-in-the-loop via durable interrupts

**What it is.** Before any state-changing action, the graph pauses at a checkpoint
persisted to Postgres, surfaces the proposed action to a human, and resumes with their
decision — surviving disconnects, restarts, and channel switches.

**Where.** The `await_approval` interrupt node and the `/resume` endpoint
([01 key flow 2](./01-architecture-overview.md), [03 §1 Pillar 5](./03-agent-platform.md)).

**Why.** An ERP agent that writes registries, sends emails, and generates invoices must be
*provably* unable to act without consent. The monolith implemented approval as an in-stream
pause (lost on disconnect); checkpointed interrupts make consent durable.

**Trade-off.** Checkpoint storage and a slightly more complex client protocol (a chat can
end in `approval_required` rather than `done`).

### 5.3 Capability allow-listing (least privilege for agents)

**What it is.** A tool existing in the catalog grants nothing: each agent's manifest
enumerates exactly which tools the model may see and call, and read/write classification
is enforced by the runtime (write ⇒ interrupt), not by prompt wording.

**Where.** The `tools:` allow-list and `kind="read"|"write"` enforcement
([03 §4](./03-agent-platform.md#4-tools-the-shared-catalog)); safety guards (host
allow-lists, tenant scoping) live inside tools so no agent can forget them.

**Why.** Prompt-level restrictions are advisory; capability-level restrictions are
structural. This is the security boundary of the agent platform.

**Trade-off.** Adding a capability is a deliberate two-step (catalog + manifest), which is
mild friction — by design.

### 5.4 Centralized model access (the Model Gateway)

**What it is.** A single service owns all LLM provider credentials and exposes a uniform
completion/embedding API; every model call in the platform flows through it and emits a
metering event as a side effect of the call itself.

**Where.** model-gateway ([02](./02-service-catalog.md)); `token.usage` events consumed by
billing-service.

**Why.** Three structural wins: provider switching is invisible to callers; the AI kill
switch and budgets act in one place; and **billing cannot be bypassed by a forgotten
callsite** — the monolith metered per-callsite, which is exactly how leaks happen.

**Trade-off.** An extra hop on the hottest path (mitigated by streaming passthrough and
co-location), and the gateway becomes critical infrastructure — replicate it early.

---

## 6. Pattern-to-document map

| Pattern | Defined in | Primary doc |
|---|---|---|
| Microservices / bounded contexts | service boundaries | [02](./02-service-catalog.md) |
| API Gateway | edge service | [01 §5](./01-architecture-overview.md) |
| Database-per-Service | data ownership rules | [01 §2](./01-architecture-overview.md), [02](./02-service-catalog.md) |
| Multi-tenancy + RLS | tenant scoping | [01 §6](./01-architecture-overview.md) |
| Hexagonal (Ports & Adapters) | service internals | [02](./02-service-catalog.md), [03 §5](./03-agent-platform.md) |
| Repository + DTO separation | adapters layer | [03 §5](./03-agent-platform.md) |
| DI / composition root | `deps.py` | [02](./02-service-catalog.md) |
| Sync HTTP + context propagation | call matrix | [01 §5–6](./01-architecture-overview.md), [02](./02-service-catalog.md) |
| Thin events + idempotent consumers | Redis Streams topics | [01 §7](./01-architecture-overview.md) |
| Private job queues | per-service workers | [01 §7](./01-architecture-overview.md), [02](./02-service-catalog.md) |
| Strangler Fig | migration phases | [05 §5](./05-migration-pros-and-cons.md) |
| Plugin discovery / convention over config | agents, tools, adapters | [03 §1](./03-agent-platform.md) |
| Declarative manifests | `manifest.yaml` | [03 §2](./03-agent-platform.md) |
| Template Method | default graph | [03 §1–2](./03-agent-platform.md) |
| ReAct loop | default graph | [03 §1](./03-agent-platform.md) |
| Human-in-the-loop interrupts | approval flow | [01 flow 2](./01-architecture-overview.md), [03](./03-agent-platform.md) |
| Capability allow-listing | tool catalog | [03 §4](./03-agent-platform.md) |
| Centralized model access | model-gateway | [02](./02-service-catalog.md) |
