# identity-service

> Source of truth for users, companies (tenants), memberships, the RBAC role/permission
> model, and trust: JWT issuance, federated + local login, JWKS publication, service tokens
> for internal calls.

**Status:** 📋 planned · **Port (dev):** 8010 · **Database:** `identity`

## Responsibilities

- Registration with email OTP verification; login; logout.
- **Authentik federation (authentication):** acts as an OIDC relying party to Authentik
  (the Industry4Z SSO) — handles the authorize/callback flow, validates the IdP token,
  upserts an `oauth_identities` link (`provider = authentik`), and resolves it to a local
  user. This is the primary login path (option C of
  [09 §3.1](../../09-industry4z-platform-integration.md#31-identity--authentik-)).
  **identity-service stays the platform token issuer** — Authentik authenticates,
  identity-service authorizes.
- JWT **RS256** access tokens (short-lived, carry user ID + company membership claims with
  the assigned role) + rotated refresh tokens stored server-side.
- Local email/password login as a fallback / non-SSO path; password hashing with **Argon2**
  (nullable for SSO-only users); password reset flow.
- Google OAuth login (identity only — Drive/Gmail scopes belong to integration-service).
- Companies (tenants): create, profile/branding, logo.
- Memberships: link users to companies with an assigned **role**; grant/revoke.
- **RBAC role & permission model** (see
  [01 §6](../../01-architecture-overview.md#authorization-model--roles--permissions-rbac)):
  - Owns the platform **permission catalog** (seeded, verb-style, grouped by domain —
    e.g. `registry.write`, `invoice.approve`, `roles.manage`); not tenant-editable.
  - **System roles** (`owner`, `co-owner`, `admin`, `member`, `viewer`) — fixed, shared.
  - **Custom roles** — tenants create their own roles and compose them from the catalog
    (gated by the `roles.manage` permission).
  - Publishes the role→permission map that `x7-common` caches so every service can resolve
    a role to its permission set locally.
- Platform-admin claim management; impersonation sessions (one-shot, read-only guard
  enforced at the gateway).
- JWKS endpoint (`/.well-known/jwks.json`) consumed by the gateway.
- Short-lived **service tokens** for internal service-to-service calls.
- Bulgarian EIK company lookup (Commercial Register provider adapters, env-gated).

## API sketch

`POST /auth/register` · `POST /auth/login` · `POST /auth/refresh` · `POST /auth/logout` ·
`POST /auth/verify` · `POST /auth/password-reset` · `GET /auth/google` + callback ·
`GET /auth/authentik/start` + `GET /auth/authentik/callback` (federated SSO) ·
`GET/PATCH /users/me` · `GET/POST /companies` · `GET/POST /companies/{id}/members` ·
`GET/POST/PATCH/DELETE /companies/{id}/roles` (custom roles) ·
`PUT /companies/{id}/members/{user_id}/role` (assign role) ·
`GET /permissions` (catalog) · `GET /companies/{id}/roles/{role_id}/permissions` ·
`GET /internal/role-permissions` (role→permission map for `x7-common`) ·
`GET /.well-known/jwks.json` · `POST /internal/service-token` ·
`GET /company-lookup?eik=`

## Data owned

`users`, `companies`, `user_companies` (membership + `role_id`), `roles` (system + custom),
`permissions` (catalog), `role_permissions`, `refresh_tokens`, `password_reset_tokens`,
`oauth_identities` (Authentik + Google), `impersonation_sessions`. External tokens encrypted
AES-256-GCM. Schema: [08 §4.1](../../08-database-architecture.md#41-identity--users-tenants-trust).

## Dependencies

| Direction | What |
|---|---|
| Called by | gateway (JWKS), platform-service, all auth flows |
| Calls | nothing (leaf service) |
| Events out | `tenant.created` (→ registry-service seeds, billing-service bonus), `user.registered`, `audit.event` |
| Jobs | expired token cleanup (daily) |

## Design notes

- Key pair rotation must be supported from day one (JWKS with `kid`).
- **Service tokens must not put this service on the hot path**: callers cache minted
  tokens until expiry and receivers verify them locally via the service-token JWKS
  (`x7-common` implements both sides) — an outage here delays token renewal, never
  in-flight requests. See [01 §6](../../01-architecture-overview.md#6-identity-tenancy-and-authorization).
- Migration: monolith already uses RS256 JWTs — Phase 1 of the strangler keeps the same
  claims shape so the monolith can verify tokens issued here
  (see [05 §5](../../05-migration-pros-and-cons.md)).

## Implementation checklist

- [ ] Schema + Alembic baseline; Argon2 hashing; RS256 keypair management
- [ ] Register/login/refresh/verify/reset flows + email via `notification.requested` events
- [ ] **Authentik OIDC federation** (authorize/callback, token validation, `oauth_identities`
      link, local-user resolution); local password as fallback
- [ ] Membership claims in access token; `X-Company-Id` validation contract with gateway
- [ ] Google OAuth login flow
- [ ] **Permission catalog** seed + `permissions` table (verb-style, domain-grouped)
- [ ] **Roles**: system roles seed + tenant custom-role CRUD (gated by `roles.manage`)
- [ ] **Role assignment** on memberships; role→permission map endpoint for `x7-common`
- [ ] JWKS endpoint + key rotation
- [ ] Service-token minting + verification helper in `libs/common`
- [ ] `tenant.created` event publication (seed default system roles for the tenant)
- [ ] EIK lookup provider adapters (port + per-provider adapter)

## References

- [01 §6 — Identity, tenancy, authorization](../../01-architecture-overview.md#6-identity-tenancy-and-authorization)
- [02 — service catalog entry](../../02-service-catalog.md#identity-service-8010)
- [07 §5.2 — dependency graph](../../07-dependency-graphs.md#52-identity-service)
