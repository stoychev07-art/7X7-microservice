# identity-service

> Source of truth for users, companies (tenants), memberships, and trust: JWT issuance,
> OAuth login, JWKS publication, service tokens for internal calls.

**Status:** 📋 planned · **Port (dev):** 8010 · **Database:** `identity`

## Responsibilities

- Registration with email OTP verification; login; logout.
- JWT **RS256** access tokens (short-lived, carry user ID + company membership claims)
  + rotated refresh tokens stored server-side.
- Password reset flow; password hashing with **Argon2**.
- Google OAuth login (identity only — Drive/Gmail scopes belong to integration-service).
- Companies (tenants): create, profile/branding, logo.
- Memberships + roles: `owner`, `co-owner`, `admin`, `member`, `viewer`; grant/revoke.
- Platform-admin claim management; impersonation sessions (one-shot, read-only guard
  enforced at the gateway).
- JWKS endpoint (`/.well-known/jwks.json`) consumed by the gateway.
- Short-lived **service tokens** for internal service-to-service calls.
- Bulgarian EIK company lookup (Commercial Register provider adapters, env-gated).

## API sketch

`POST /auth/register` · `POST /auth/login` · `POST /auth/refresh` · `POST /auth/logout` ·
`POST /auth/verify` · `POST /auth/password-reset` · `GET /auth/google` + callback ·
`GET/PATCH /users/me` · `GET/POST /companies` · `GET/POST /companies/{id}/members` ·
`GET /.well-known/jwks.json` · `POST /internal/service-token` ·
`GET /company-lookup?eik=`

## Data owned

`users`, `companies`, `user_companies`, `refresh_tokens`, `password_reset_tokens`,
`oauth_identities`, `impersonation_sessions`. External tokens encrypted AES-256-GCM.

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
- [ ] Membership claims in access token; `X-Company-Id` validation contract with gateway
- [ ] Google OAuth login flow
- [ ] JWKS endpoint + key rotation
- [ ] Service-token minting + verification helper in `libs/common`
- [ ] `tenant.created` event publication
- [ ] EIK lookup provider adapters (port + per-provider adapter)

## References

- [01 §6 — Identity, tenancy, authorization](../../01-architecture-overview.md#6-identity-tenancy-and-authorization)
- [02 — service catalog entry](../../02-service-catalog.md#identity-service-8010)
- [07 §5.2 — dependency graph](../../07-dependency-graphs.md#52-identity-service)
