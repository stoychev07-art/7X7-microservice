# x7-common — the shared kernel

> One small, dependency-light package (`libs/common`, import name `x7_common`) installed
> by every service. It owns the boring-but-critical plumbing so that all 11 services boot,
> log, authenticate, call each other, and publish events the exact same way.

**Status:** 📋 planned · **Location:** `libs/common` · **Distribution:** `x7-common` ·
**Import:** `x7_common`

## The contract

Two rules make a shared kernel safe instead of a coupling trap:

1. **No business logic, ever.** If a function knows what an invoice, a registry, or an
   agent is, it does not belong here. Only transport, observability, auth plumbing, and
   cross-service DTOs.
2. **Additive evolution.** Every service depends on this package, so a breaking change is
   an 11-service change. New fields/helpers are fine; renames and removals need a
   deprecation window.

Services may **never** import another service's code — `x7_common` is the only shared
Python dependency between them.

## Package layout

```
libs/common/
├── x7_common/
│   ├── __init__.py
│   ├── config.py          # CommonSettings — base for every service's Settings
│   ├── auth.py            # AuthContext from gateway headers · service tokens · require_role
│   ├── http.py            # process-wide pooled httpx.AsyncClient
│   ├── middleware.py      # RequestIdMiddleware (read/generate/echo X-Request-Id)
│   ├── logging.py         # structlog JSON setup, request_id/tenant contextvars
│   ├── telemetry.py       # OpenTelemetry init (FastAPI + httpx instrumentation)
│   ├── health.py          # health_router(checks) → /health + /ready
│   ├── lifespan.py        # composable lifespan (httpx pool + service hooks)
│   ├── events.py          # Redis Streams EventBus: publish / consumer groups
│   ├── errors.py          # error envelope + exception handlers (uniform 4xx/5xx shape)
│   ├── pagination.py      # cursor pagination params + response wrapper
│   ├── schemas/           # cross-service DTOs (the wire contracts)
│   │   ├── auth.py        #   AuthContext, Role
│   │   ├── chat.py        #   ChatMessage, ToolSpec, ToolCall, Usage, StreamChunk,
│   │   │                  #   CompletionRequest/Response, EmbeddingsRequest/Response
│   │   ├── retrieval.py   #   Chunk, RetrieveRequest/Response
│   │   └── events.py      #   typed event payloads (TenantCreated, TokenUsage, …)
│   └── testing/
│       └── fixtures.py    # app factory fixture, fake auth headers, event-bus stub
├── tests/
└── pyproject.toml
```

## Module guide

### `config.py` — `CommonSettings`

Every service's `Settings` subclasses this; all configuration is env-driven (12-factor).

```python
class CommonSettings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    service_name: str = "x7-service"
    environment: str = "local"
    log_level: str = "INFO"

    # Internal addressing — Docker DNS names; overridden per environment
    gateway_url: str = "http://gateway:8000"
    identity_url: str = "http://identity-service:8000"
    agent_url: str = "http://agent-service:8000"
    model_gateway_url: str = "http://model-gateway:8000"
    knowledge_url: str = "http://knowledge-service:8000"
    registry_url: str = "http://registry-service:8000"
    business_url: str = "http://business-service:8000"
    document_url: str = "http://document-service:8000"
    billing_url: str = "http://billing-service:8000"
    integration_url: str = "http://integration-service:8000"
    platform_url: str = "http://platform-service:8000"

    # Redis (cache, queues, event streams)
    redis_url: str = "redis://redis:6379/0"

    # Service-to-service auth
    service_token_audience: str = ""        # set to the service's own name
    service_token_ttl_seconds: int = 60

    # Telemetry
    otel_enabled: bool = False
    otel_exporter_otlp_endpoint: str | None = None
```

A service adds only its own fields (`database_url`, `stripe_secret`, …) in its `config.py`.

### `auth.py` — identity plumbing (no policy)

Key difference from a bearer-decoding library: in this architecture **the gateway verifies
the end-user JWT** (RS256 via JWKS) and forwards verified headers. Services therefore
*extract*, not *verify*, user identity — but they **do** verify service tokens, which prove
the call came from inside the platform.

```python
@dataclass(frozen=True, slots=True)
class AuthContext:
    user_id: str
    company_id: str          # the active tenant
    roles: list[str]         # roles within that company
    is_platform_admin: bool
    request_id: str

def auth_context(request: Request) -> AuthContext: ...
    # FastAPI dependency — reads X-User-Id / X-Company-Id / X-Roles / X-Request-Id;
    # 401 if missing (a request that bypassed the gateway)

def require_role(*allowed: str) -> Depends: ...
    # require_role("owner", "co-owner") — 403 below threshold; 404 for
    # platform-admin-only routes (ADR-015 behavior)

def require_service_token(audience: str) -> Depends: ...
    # verifies X-Service-Token minted by identity-service (iss/aud/exp);
    # applied to /internal/* routes
```

Service-token mechanics — designed to keep identity-service **off the per-request path**:
the client helper mints a token once and caches it until shortly before `exp`, refreshing
in the background; `require_service_token` verifies the signature **locally** against
identity-service's service-token JWKS (cached, with the same `kid` rotation handling as
the user JWKS). An identity-service outage therefore blocks only token renewal — and only
once cached tokens expire — never in-flight traffic.

### `http.py` — one pooled `httpx.AsyncClient` per process

Singleton with sane timeouts and connection limits; initialized/closed by the lifespan.
All adapters use `get_client()` — nobody constructs ad-hoc clients (connection churn on
the agent tool path is a real latency cost).

### `events.py` — the Redis Streams event bus

The piece mango-common doesn't have, required by our event-driven flows
([01 §7](../../01-architecture-overview.md#7-asynchronous-work-and-events)):

```python
bus = EventBus(redis_url, service_name)

await bus.publish("token.usage", TokenUsage(tenant_id=..., tokens=..., feature=...))

@bus.consumer("tenant.created", group="registry-service")
async def on_tenant_created(evt: TenantCreated) -> None:
    ...  # idempotent — at-least-once delivery
```

- Typed payloads from `schemas/events.py` (Pydantic-validated on both ends).
- Consumer groups per service; automatic ack on success, retry with dead-letter after N
  failures.
- Every event carries `event_id`, `occurred_at`, `tenant_id`, `request_id` — consumers use
  `event_id` as their idempotency key.

### `logging.py` + `middleware.py` + `telemetry.py` — observability

- structlog JSON lines with `service`, `request_id`, `user_id`, `company_id` bound from
  contextvars — set once by `RequestIdMiddleware` + `auth_context`, present on every line.
- `RequestIdMiddleware`: read `X-Request-Id` (or generate), bind, echo on response.
- `telemetry.init(app)`: OpenTelemetry tracing for FastAPI + httpx, so a trace started at
  the gateway continues through every hop.

### `health.py` + `lifespan.py` — uniform ops surface

- `health_router(checks={"db": ping_db, "redis": ping_redis})` mounts `GET /health`
  (liveness, always 200) and `GET /ready` (503 if any probe fails) — the contract Docker
  Compose / k8s checks rely on for every service.
- `make_lifespan(on_startup=[...], on_shutdown=[...])` boots the httpx pool, telemetry,
  and the event bus, then runs service-specific hooks (DB engine, arq pool).

### `schemas/` — the wire contracts

Only DTOs that **cross a service boundary** live here (chat/completion shapes used by
agent-service ↔ model-gateway, retrieval chunks, event payloads, `AuthContext`). A DTO
used by exactly one service stays in that service's `models/` — e.g. session/message
DTOs live in agent-service, since conversations are internal to it. Moving a type here
is a deliberate act: it becomes a frozen contract.

### `errors.py` + `pagination.py` — API consistency

- One error envelope (`{"error": {"code", "message", "request_id"}}`) and shared exception
  handlers, so all 11 services fail identically and the frontend handles one shape.
- Cursor pagination params/response used by every list endpoint.

### `testing/` — fixtures every service reuses

App-factory fixture with lifespan, `auth_headers(user, company, roles)` for simulating
gateway-forwarded identity, an in-memory `EventBus` stub asserting published events, and a
disposable-Postgres helper (testcontainers).

## What deliberately does NOT belong here

| Tempting to share | Why it stays out |
|---|---|
| SQLAlchemy base / repositories | Database-per-service: each service owns its persistence shape |
| Domain enums (invoice status, registry roles) | Business logic — belongs to the owning service's contract |
| Service client classes (`RegistryClient`, …) | Callers define *ports* for what *they* need (hexagonal rule); a shared client would couple every consumer to one shape |
| LLM/provider SDK wrappers | model-gateway's adapters only |
| Feature flags / kill switches | model-gateway and owning services |

## Dependencies & tooling


```toml
[project]
name = "x7-common"
requires-python = ">=3.12"
dependencies = [
  "fastapi>=0.111", "httpx>=0.27",
  "pydantic>=2.7", "pydantic-settings>=2.3",
  "structlog>=24", "pyjwt>=2.8",
  "redis>=5",                       # event bus + cache
  "opentelemetry-sdk>=1.25",
  "opentelemetry-instrumentation-fastapi>=0.46b0",
  "opentelemetry-instrumentation-httpx>=0.46b0",
]
[project.optional-dependencies]
dev = ["pytest>=8", "pytest-asyncio>=0.23", "ruff>=0.5", "mypy>=1.10", "testcontainers>=4"]
```

Installed by each service as a `uv` workspace path dependency
(`x7-common = { workspace = true }`), so changes are picked up locally without publishing;
CI runs the common test suite before any service suite.

## Implementation checklist

- [ ] Package scaffold + pyproject + uv workspace wiring
- [ ] `CommonSettings`, `http`, `lifespan`, `health` (smallest useful core)
- [ ] `RequestIdMiddleware` + structlog setup + contextvars
- [ ] `AuthContext` dependency + `require_role` + service tokens (with identity-service)
- [ ] `EventBus` publish/consume + dead-letter + typed event payloads
- [ ] Error envelope + exception handlers; pagination helpers
- [ ] `schemas/` chat + retrieval DTOs (agent ↔ model-gateway contract first)
- [ ] `testing/` fixtures; common test suite in CI ahead of service suites
- [ ] OpenTelemetry init verified end-to-end through the gateway

## References

- [02 — Service Catalog (shared `libs/common` rules)](../../02-service-catalog.md)
- [01 §7 — events](../../01-architecture-overview.md#7-asynchronous-work-and-events) ·
  [01 §10 — cross-cutting standards](../../01-architecture-overview.md#10-cross-cutting-standards-every-service)
- [06 — patterns: DI/composition root, thin events](../../06-architectural-patterns.md)
