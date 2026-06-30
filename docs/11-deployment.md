# 11 — Deployment & Operations

The operational source of truth for running the platform: the single image and how it
becomes many services, configuration and secrets, the data plane and its initialization,
migrations, networking, health/observability, the dev and production topologies, scaling,
the procedure for splitting a service out, backups, and CI/CD.

> **Relationship to other docs.** [01 §9](./01-architecture-overview.md#9-deployment--environments)
> states the deployment principles; [10-phase1-co-deployment.md](./10-phase1-co-deployment.md)
> explains *why* the 13 logical services group into ~5 deployables and when to split them.
> **This doc is the *how*** — the concrete artifacts. Where the two overlap (the compose file,
> the grouping table), **this document is canonical**.

---

## 1. The deployment model in one picture

One Git repo → **one Docker image** → run it many times, each instance told which services to
host via a single `SERVICES` environment variable. A thin gateway is the only publicly exposed
container; everything else lives on an internal network.

```mermaid
flowchart TB
    repo["one repo<br/>services/* + app/ entrypoint"] --> img["one image<br/>(uv-built, all service packages)"]

    img --> gw["gateway<br/>SERVICES=gateway"]
    img --> idn["identity (+worker)<br/>SERVICES=identity"]
    img --> ai["ai-plane (+worker)<br/>SERVICES=agent,model-gateway,knowledge"]
    img --> erp["erp-core (+worker)<br/>SERVICES=registry,business,document"]
    img --> ops["ops-plane (+worker)<br/>SERVICES=billing,integration,platform"]

    subgraph net["internal network (not publicly reachable)"]
        idn
        ai
        erp
        ops
    end

    clients["clients"] ==> gw ==> net

    style gw fill:#e8f0fe,stroke:#3b6fc9
    style ai fill:#fff3d6,stroke:#cc9a06
```

The same image runs in dev (Docker Compose) and prod (Compose or any orchestrator). Only the
**number of replicas**, the **`SERVICES` split**, and the **endpoints/secrets** differ between
environments — never the code.

---

## 2. The single image

All services build from one `Dockerfile` at the repo root. `uv` installs the workspace
(every `services/*` package + `libs/common`), so the image contains all service code; the
runtime decides which parts to activate.

```dockerfile
# Dockerfile (repo root) — one image for every deployable
FROM python:3.12-slim AS base
RUN pip install --no-cache-dir uv
WORKDIR /app

# install deps first for layer caching
COPY pyproject.toml uv.lock ./
COPY libs/ ./libs/
COPY services/ ./services/
COPY app/ ./app/                      # shared entrypoint (registry, main, workers)
RUN uv sync --frozen --no-dev

EXPOSE 8000
# default command = web server; worker containers override `command`
CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Rationale for one image (vs. one per service):

- **CI builds and caches once**, not 13×; path-filtered tests still run per service.
- The co-deployment grouping becomes a pure **runtime** choice (`SERVICES`), so regrouping or
  splitting never requires a new image.
- Image size cost is marginal — all services share the same heavy deps (FastAPI, SQLAlchemy,
  httpx, LangGraph) anyway.

> When a service is eventually split to its own scaling unit, it still uses **this same image**
> with `SERVICES` set to just that one service. A per-service image is an optimization you may
> never need.

---

## 3. How `SERVICES` is wired (the mechanism)

This is the knob every deployable turns. Three pieces in the shared `app/` package.

### 3.1 Every service exports the same shape

By convention each service package exposes a router, optional worker settings, and an optional
`register()` hook:

```python
# services/billing/app/__init__.py   (identical pattern in every service)
from fastapi import APIRouter

router = APIRouter(prefix="/api/v1/billing", tags=["billing"])
worker_settings = WorkerSettings        # arq functions + cron (omit if no background work)

def register(app):                      # optional: event consumers, lifespan, startup checks
    ...
```

### 3.2 A registry maps service name → package

```python
# app/registry.py  (in the shared image; knows all services)
SERVICE_MODULES = {
    "gateway":       "services.gateway.app",
    "identity":      "services.identity.app",
    "agent":         "services.agent.app",
    "model-gateway": "services.model_gateway.app",
    "knowledge":     "services.knowledge.app",
    "registry":      "services.registry.app",
    "business":      "services.business.app",
    "document":      "services.document.app",
    "billing":       "services.billing.app",
    "integration":   "services.integration.app",
    "platform":      "services.platform.app",
    # "iot" / "manufacturing" added when those verticals ship
}
```

### 3.3 The web entrypoint mounts only the selected routers

```python
# app/main.py  (default container command runs this)
import os, importlib
from fastapi import FastAPI
from app.registry import SERVICE_MODULES

def create_app() -> FastAPI:
    app = FastAPI()
    for name in os.environ["SERVICES"].split(","):     # e.g. billing,integration,platform
        mod = importlib.import_module(SERVICE_MODULES[name.strip()])
        app.include_router(mod.router)                 # mount that service's HTTP routes
        if hasattr(mod, "register"):
            mod.register(app)                          # event consumers, lifespan hooks
    return app

app = create_app()
```

A container booted with `SERVICES=billing,integration,platform` then serves, in **one process**:

```
/api/v1/billing/**      /api/v1/integration/**      /api/v1/platform/**
```

### 3.4 The worker entrypoint registers only the selected jobs

```python
# app/workers.py  (worker containers run: arq app.workers.WorkerSettings)
import os, importlib
from app.registry import SERVICE_MODULES

_functions, _cron = [], []
for name in os.environ["SERVICES"].split(","):
    mod = importlib.import_module(SERVICE_MODULES[name.strip()])
    ws = getattr(mod, "worker_settings", None)
    if ws:
        _functions += ws.functions
        _cron += ws.cron_jobs

class WorkerSettings:           # what arq executes
    functions = _functions
    cron_jobs = _cron
    # redis_settings injected from env (shared common.Settings)
```

This is the same **discover-and-mount-by-name** convention the platform already uses for agents
and integration adapters ([01 §2.3](./01-architecture-overview.md#2-architectural-principles)) —
here applied to whole services, driven by config.

### 3.5 What "local call" then means

Because the co-located services are objects in the **same process**, a call between them
(`platform → billing`, `business → registry`, `agent → model-gateway`) can be:

- **loopback HTTP** — the existing httpx adapter points at `127.0.0.1`; no network, no second
  container; or
- **in-process** — because the caller depends on a `Protocol` port, `deps.py` can inject an
  adapter that calls the co-located domain function directly (no serialization). Reserve this
  for the hottest hop, `agent → model-gateway`.

Splitting later flips that adapter back to a remote URL — no domain-code change. See
[10 §3](./10-phase1-co-deployment.md#3-what-each-internal-call-becomes).

---

## 4. Service → deployable map

| Deployable | `SERVICES` value | Worker? | Publicly exposed | Notes |
|---|---|---|---|---|
| `gateway` | `gateway` | no | **yes** (only one) | Stateless edge; scale replicas first |
| `identity` | `identity` | yes | no | Security-critical leaf; isolated |
| `ai-plane` | `agent,model-gateway,knowledge` | yes | no | The streaming hot path |
| `erp-core` | `registry,business,document` | yes | no | Structured ERP domain |
| `ops-plane` | `billing,integration,platform` | yes | no | Event consumers + connectivity |
| `iot` *(deferred)* | `iot` | yes | no | Start split; needs TimescaleDB |
| `manufacturing` *(deferred)* | `manufacturing` | yes | no | Start split |

Grouping rationale and split triggers: [10 §2](./10-phase1-co-deployment.md#2-the-recommended-grouping)
and [10 §5](./10-phase1-co-deployment.md#5-when-to-split-each-group-back-out).

---

## 5. Configuration & secrets

12-factor: all config via environment variables, one `Settings(BaseSettings)` per service
extending a shared `common.Settings` ([01 §9](./01-architecture-overview.md#9-deployment--environments)).

| Variable (illustrative) | Used by | Example |
|---|---|---|
| `SERVICES` | every app container | `agent,model-gateway,knowledge` |
| `DATABASE_URL` | each service (its own schema) | `postgresql+asyncpg://agent:***@postgres/agent` |
| `REDIS_URL` | all | `redis://redis:6379/0` |
| `JWKS_URL` / `IDENTITY_URL` | gateway, all services | `http://identity:8000/.well-known/jwks.json` |
| `<SERVICE>_URL` (per port) | callers | `MODEL_GATEWAY_URL=http://127.0.0.1:8000` (loopback when co-located) |
| `LLM_ADAPTER` | agent, model-gateway | `httpx` \| `inprocess` (selects the port adapter) |
| `S3_ENDPOINT` / keys | knowledge, document | MinIO (dev) / S3 (prod) |
| Provider secrets | one owner each | `STRIPE_*`→billing, `NANGO_*`/`DOCUMENSO_*`→integration, `BREVO_*`/`TELEGRAM_*`→platform, LLM keys→model-gateway |

Rules:

- **Secrets never in code or images.** Dev: a git-ignored `.env`. Prod: a secret manager
  injected as env at deploy.
- **One owner per external credential** (the §[07 rule 4](./07-dependency-graphs.md#6-dependency-rules-the-invariants-behind-the-graphs)):
  only model-gateway holds LLM keys, only billing holds Stripe keys, etc. Co-location does not
  change ownership — a key belongs to its service even when sharing a process.
- **Per-service DB credentials**: each service connects with its own role (see §6), so
  `DATABASE_URL` differs per service even on a shared cluster.

---

## 6. Data plane

Database-per-service holds logically; physically, phase-1 runs **one PostgreSQL 16 cluster
with one logical database + one scoped role per service**. This cuts operational surface
without weakening the boundary (no cross-schema joins; enforced by role grants).

```mermaid
flowchart LR
    subgraph pg["one PostgreSQL 16 cluster"]
        d1[("identity")]; d2[("agent")]; d3[("modelgw")]
        d4[("knowledge")]; d5[("registry")]; d6[("business")]
        d7[("document")]; d8[("billing")]; d9[("integration")]
        d10[("platform")]
    end
    ts[("TimescaleDB cluster<br/>iot (deferred)")]
    qd[("Qdrant<br/>vectors")]
    rd[("Redis 7<br/>cache · arq · streams (AOF)")]
    s3[("S3 / MinIO<br/>files · artifacts")]

    ai["ai-plane"] --> d2 & d3 & d4 & qd & s3 & rd
    erp["erp-core"] --> d5 & d6 & d7 & s3 & rd
    ops["ops-plane"] --> d8 & d9 & d10 & rd
    identity["identity"] --> d1 & rd
    gateway["gateway"] --> rd
```

### 6.1 Database initialization (one DB + role per service)

The `postgres` container runs an init script that creates each logical DB and a role that can
**only** see its own schema — the mechanism that keeps "no cross-schema joins" honest even on a
shared cluster.

```bash
#!/bin/bash
# infra/pg-init/01-databases.sh — runs once on first postgres boot
set -euo pipefail
IFS=',' read -ra DBS <<< "$POSTGRES_MULTIPLE_DBS"
for db in "${DBS[@]}"; do
  psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" <<-EOSQL
    CREATE DATABASE "$db";
    CREATE ROLE "${db}_svc" LOGIN PASSWORD '${db}_pw';   -- prod: inject real secrets
    GRANT ALL PRIVILEGES ON DATABASE "$db" TO "${db}_svc";
    -- ${db}_svc connects only to its own DB; it has no grant on any other DB.
EOSQL
done
```

- **Qdrant** (vectors for knowledge-service) and **Redis** (cache, arq queues, event streams
  with AOF persistence) are single shared clusters.
- **Object storage**: MinIO in dev, any S3-compatible store in prod; bucket-per-owning-service.
- **iot** uses a **separate TimescaleDB cluster** (Postgres + extension) — stood up only when
  the IoT vertical is enabled.

---

## 7. Migrations

Each service owns its Alembic history; co-location does not merge them — there are **13 (or
the live count of) independent migration histories**, one per schema.

- **Run on deploy, before new code serves traffic** ([01 §10](./01-architecture-overview.md#10-cross-cutting-standards-every-service)).
- A small `migrate` job (same image) runs `alembic upgrade head` for each service in its
  deployable before the `web`/`worker` containers start:

```yaml
  migrate:
    build:
      context: .
    env_file:
      - .env
    depends_on:
      - postgres
    command: ["python", "-m", "app.migrate", "--services", "agent,model-gateway,knowledge"]
    restart: "no"
```

`app.migrate` iterates the named services and runs each one's `alembic upgrade head` against
its own `DATABASE_URL`. Keep migrations **backward-compatible** (expand/contract) so a rolling
deploy never breaks the still-running old code.

---

## 8. Networking & the gateway

- **Single entry point.** Only the `gateway` container publishes a port. Every other service is
  reachable only on the internal Docker/orchestrator network — never from the public internet
  ([01 §5](./01-architecture-overview.md#5-the-api-gateway)).
- **Routing table = path prefix → upstream host.** The gateway maps `/api/v1/<prefix>/**` to a
  deployable. Because several prefixes can resolve to the same host (e.g. `agent`, `model-gateway`,
  `knowledge` all → `ai-plane`), the routing table — not the process count — is the source of
  truth for "where does this prefix go".

```python
# gateway routing table (env-driven) — phase-1 values
ROUTES = {
    "auth": "http://identity:8000",        "users": "http://identity:8000",
    "companies": "http://identity:8000",   "memberships": "http://identity:8000",
    "agents": "http://ai-plane:8000",      "sessions": "http://ai-plane:8000",
    "knowledge": "http://ai-plane:8000",   "v1": "http://ai-plane:8000",   # model-gateway
    "registries": "http://erp-core:8000",  "clients": "http://erp-core:8000",
    "invoices": "http://erp-core:8000",    "documents": "http://erp-core:8000",
    "billing": "http://ops-plane:8000",    "integrations": "http://ops-plane:8000",
    "notifications": "http://ops-plane:8000", "support": "http://ops-plane:8000",
    # "iot": "http://iot:8000", "production-orders": "http://manufacturing:8000",
}
```

- **TLS terminates at the gateway** (or a reverse proxy in front of it). Internal hops use
  service tokens over the private network ([01 §6](./01-architecture-overview.md#6-identity-tenancy-and-authorization)).
- Webhooks (Stripe, Brevo, Documenso, Nango) pass through the gateway with raw body preserved
  and are signature-verified by the owning service.

---

## 9. Health, readiness & observability

Every service exposes the standard endpoints ([01 §10](./01-architecture-overview.md#10-cross-cutting-standards-every-service));
in a co-deployed container the app aggregates them across mounted services:

- `GET /health` — liveness (process up).
- `GET /ready` — readiness (its DB + Redis reachable; gateway/orchestrator gates traffic on this).
- **Structured JSON logs** (structlog) with `request_id`, `user_id`, `company_id`, `service` on
  every line — the `service` field still distinguishes co-located services in one log stream.
- **OpenTelemetry traces** — the gateway starts the trace; each hop (network *or* loopback)
  continues it, so a chat turn is one trace even when several spans share a process.
- **Sentry** for errors.

> Observability matters *more* under co-deployment, not less: when two services share a process,
> the `service` log field and the trace span name are how you tell their behavior apart.

---

## 10. Dev environment

One command boots the world: `docker compose up`. The full compose file (one Postgres with 13
logical DBs, Redis, Qdrant, MinIO, the internal backers, and the ~5 app deployables each with a
web + worker container) is maintained in
[10 §7](./10-phase1-co-deployment.md#7-sample-dev-docker-compose-sketch) — that is the working
reference; this section explains how to drive it.

```bash
cp .env.example .env            # fill secrets (Stripe test keys, etc.)
docker compose up --build       # builds the single image, starts everything
docker compose run --rm migrate # or let each deployable's migrate job run on start
docker compose logs -f ai-plane # follow one deployable
```

The Next.js dev server runs alongside (talks only to the gateway at `:8000`).

---

## 11. Production topology

### Phase 1 — single host

Same images, same compose shape, on one adequately-sized host. The architecture does **not**
require Kubernetes to be correct.

- Data plane: **standalone PostgreSQL 16** (managed instance recommended) + an S3-compatible
  object store; Redis with AOF + a replica.
- TLS at the gateway; secrets from a manager; backups scheduled (§13).

### Scale-out path (in order)

```mermaid
flowchart LR
    s0["single host<br/>~5 deployables"] --> s1["replicas of gateway"]
    s1 --> s2["replicas of ai-plane<br/>(+ separate worker scaling)"]
    s2 --> s3["split model-gateway / knowledge<br/>out of ai-plane"]
    s3 --> s4["split erp-core / ops-plane<br/>members as load demands"]
```

1. **Gateway replicas** behind a load balancer (stateless — trivial).
2. **ai-plane replicas** — the chat/LLM hot path; scale web and worker independently.
3. **Split the hottest member out** (model-gateway or knowledge) when contention shows.
4. **Split erp-core / ops-plane members** as their individual load or team ownership demands.

Each step is a **config/topology change** (replica count, `SERVICES` value, routing table) on
the same image — see §12.

---

## 12. Splitting a service out (the procedure)

When a [split trigger](./10-phase1-co-deployment.md#5-when-to-split-each-group-back-out) fires —
say, peel `knowledge` out of `ai-plane`:

1. **Add a new deployable** (same image) with `SERVICES=knowledge` (+ its worker), `depends_on`
   its infra (Qdrant, MinIO).
2. **Remove `knowledge`** from `ai-plane`'s `SERVICES` value (now `agent,model-gateway`).
3. **Give it its own connection string** — it already had its own `knowledge` schema + role, so
   this is just pointing `DATABASE_URL` at the same DB from the new container.
4. **Flip the adapter**: set the callers' `KNOWLEDGE_URL` / adapter mode from loopback/in-process
   to `http://knowledge:8000`. Because callers depend on `RetrievalPort`, no domain code changes.
5. **Update the gateway routing table**: `knowledge → http://knowledge:8000`.
6. **Migrations, events, secrets unchanged** — the schema, its Alembic history, the Redis
   streams, and any credentials move with the service automatically (they were never shared).

No rewrite, no data migration. This is the payoff for keeping code + data boundaries split even
while co-deploying.

---

## 13. Backups & durability

| Asset | Mechanism |
|---|---|
| PostgreSQL (all schemas) | Scheduled `pg_dump`/PITR (managed-instance snapshots in prod); one cluster, but back up per-DB so a restore can be service-scoped |
| Redis | AOF persistence + a replica; queued events/jobs survive restart ([01 §7](./01-architecture-overview.md#7-asynchronous-work-and-events)) |
| Object storage | S3/MinIO versioning + lifecycle; originals and generated artifacts |
| Critical events (`token.usage`) | **Transactional outbox** in the producer — the domain row and pending event commit together; losing the event requires losing the DB, not just Redis |
| External backers | Nango (own pg/redis), Documenso (own pg) — back up their stores too if self-hosted |

Cutover/migration durability (strangler phases, per-tenant read-back verification) is covered in
[05 §5](./05-migration-pros-and-cons.md#5-risk-reducing-migration-strategy-strangler-not-big-bang).

---

## 14. CI/CD

- **Build the single image once**; tag per commit.
- **Per-service test + lint, triggered by path filters** — touching `services/billing/**` runs
  billing's suite against a disposable Postgres (testcontainers), not the whole monorepo.
- **Contract tests** validate each service's OpenAPI spec is backward-compatible before deploy.
- **Deploy** = push the new image tag, run the `migrate` job (expand/contract), then roll the
  `web`/`worker` containers. Rolling/zero-downtime works because migrations are
  backward-compatible and the gateway gates on `/ready`.

---

## 15. Quick reference

| Task | Command |
|---|---|
| Boot everything (dev) | `docker compose up --build` |
| Run migrations | `docker compose run --rm migrate` |
| Follow one deployable | `docker compose logs -f ai-plane` |
| Scale the gateway (prod Compose) | `docker compose up -d --scale gateway=3` |
| Add a service to a deployable | edit its `SERVICES` env, redeploy that container |
| Split a service out | §12 procedure |

---

## References

- [01 §5 gateway](./01-architecture-overview.md#5-the-api-gateway) ·
  [01 §9 deployment](./01-architecture-overview.md#9-deployment--environments) ·
  [01 §10 cross-cutting](./01-architecture-overview.md#10-cross-cutting-standards-every-service)
- [10 — phase-1 co-deployment topology](./10-phase1-co-deployment.md) (grouping rationale + the dev compose)
- [07 — dependency graphs](./07-dependency-graphs.md) (incl. §4.5 co-deployment overlay)
- [05 §5 — migration strategy](./05-migration-pros-and-cons.md#5-risk-reducing-migration-strategy-strangler-not-big-bang)
- [08 — database architecture](./08-database-architecture.md)
