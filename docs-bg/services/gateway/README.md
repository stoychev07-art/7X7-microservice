# gateway

> The single public entry point. Owns cross-cutting edge concerns — routing, JWT
> verification, rate limiting, streaming passthrough — and **zero business logic**.

**Status:** 📋 planned · **Port (dev):** 8000 · **Database:** none

## Responsibilities

- Reverse-proxy all client traffic to internal services by path prefix
  (`/api/v1/auth/* → identity-service`, `/api/v1/agents/* → agent-service`, …).
- Verify JWT signatures against identity-service's JWKS (cached); reject unauthenticated
  requests except public routes (login, register, webhooks, health).
- Inject verified context headers downstream: `X-User-Id`, `X-Company-Id`, `X-Roles`,
  `X-Request-Id` — and strip any client-supplied values of those headers.
- Validate the requested `X-Company-Id` against the token's membership claims.
- Redis-backed rate limiting per route class (auth, chat, default, webhooks).
- Transparent SSE/chunked passthrough for chat streams (no buffering).
- Raw-body preservation for webhook routes (Stripe, Brevo) — the owning service verifies
  signatures itself.
- CORS, body-size limits, IP allow-list for `/admin` routes.
- **Impersonation read-only guard**: requests carrying an impersonation claim may only
  use safe methods (GET/HEAD); mutating requests are rejected at the edge. The sessions
  themselves live in identity-service — the gateway only enforces the guard.

## Explicitly out of scope

Aggregation, response transformation, any domain logic. If a client needs an aggregate
view, the owning service exposes the endpoint.

## Routing table (initial)

| Prefix | Target |
|---|---|
| `/api/v1/auth`, `/api/v1/users`, `/api/v1/companies` | identity-service |
| `/api/v1/agents`, `/api/v1/sessions`, `/api/v1/skills`, `/api/v1/memory` | agent-service |
| `/api/v1/knowledge` | knowledge-service |
| `/api/v1/registries`, `/api/v1/clients`, `/api/v1/dashboard` | registry-service |
| `/api/v1/business` (invoices, stock, expenses) | business-service |
| `/api/v1/documents`, `/api/v1/prices`, `/api/v1/margins`, `/api/v1/kss` | document-service |
| `/api/v1/billing`, `/api/v1/webhooks/stripe` | billing-service |
| `/api/v1/integrations` | integration-service |
| `/api/v1/notifications`, `/api/v1/webhooks/brevo`, `/api/v1/support`, `/api/v1/admin` | platform-service (+ per-service admin routes) |

During the strangler migration, unmigrated prefixes point at the legacy monolith — the
routing table *is* the migration state (see [05 §5](../../05-migration-pros-and-cons.md)).

## Implementation checklist

- [ ] FastAPI app + httpx-based proxy with connection pooling and streaming support
- [ ] JWKS fetch + cache + key-rotation handling
- [ ] Context-header injection/stripping middleware
- [ ] Membership-claim validation for `X-Company-Id`
- [ ] Redis rate limiter (one implementation, bucket config per route class)
- [ ] SSE passthrough verified end-to-end with agent-service
- [ ] Webhook raw-body routes
- [ ] Impersonation read-only guard (block mutating methods when the token carries an
      impersonation claim)
- [ ] OpenTelemetry trace start + `X-Request-Id` generation
- [ ] Routing table as config (env/yaml), not code — needed for strangler flips

## References

- [01 §5 — The API Gateway](../../01-architecture-overview.md#5-the-api-gateway)
- [01 §6 — auth flow diagram](../../01-architecture-overview.md#6-identity-tenancy-and-authorization)
- [06 §1.2 — API Gateway pattern](../../06-architectural-patterns.md)
- [07 §5.1 — dependency graph](../../07-dependency-graphs.md#51-gateway)
