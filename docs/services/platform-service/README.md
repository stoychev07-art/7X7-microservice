# platform-service

> Platform plumbing in one place: in-app notifications, transactional email, ops alerts,
> support tickets, the central audit sink, and platform settings. Four small modules —
> `notifications/`, `support/`, `audit/`, `settings/` — deliberately one service: each is
> low-traffic, mostly event-consuming, and has no domain coupling, so separate deployables
> would buy nothing but extra databases, CI pipelines, and dashboards.

**Status:** 📋 planned · **Port (dev):** 8090 · **Database:** `platform`

## Responsibilities

### Notifications (`notifications/`)
- In-app notifications (bell): create, list, mark-read; per-user preferences.
- Transactional email via provider adapters: **Brevo** (production) and **log** (dev
  fallback); template rendering with i18n (bg/en).
- Brevo delivery webhook → delivery status tracking.
- Ops alerts to the Telegram channel (platform operators, not users).
- Consumes `notification.requested` from every service — plus domain events it turns into
  notifications (`invoice.issued`, `stock.low`).

### Support (`support/`)
- Support tickets: tenant-side create/reply, staff-side queue + replies; updates notify
  via the notifications module (in-process, no event round-trip needed).

### Audit (`audit/`)
- Central **audit-log sink**: consumes `audit.event` from every service into one queryable
  trail; retention policies (e.g. 90d audit, 30d traces — configurable).

### Settings (`settings/`)
- Platform settings (global flags and defaults).
- Admin aggregation endpoints **only where unavoidable** — each domain service owns its
  own admin endpoints (tokens admin in billing-service, provider admin in model-gateway,
  etc.); this service does not become a god-service.

## API sketch

`GET /notifications` · `POST /notifications/read` · `GET/PATCH /preferences` ·
`POST /webhooks/brevo` · `GET/POST /support/tickets` ·
`POST /support/tickets/{id}/messages` · `GET /audit?filter=` · `GET/POST /settings`

## Data owned

`notifications`, `deliveries`, `preferences`, `support_tickets`, `support_messages`,
`audit_log`, `platform_settings`.

## Dependencies

| Direction | What |
|---|---|
| Called by | gateway (bell UI, support UI, `/admin` zone of the Next.js app) |
| Calls | Brevo API, Telegram Bot API (ops channel), identity-service (user/tenant lookups), billing-service (usage views) |
| Events in | `notification.requested`, `invoice.issued`, `stock.low`, `audit.event` |
| Jobs | email send queue with retry/backoff, audit/trace retention |

## Design notes

- Every other service sends email **only** by publishing `notification.requested` — no
  service holds SMTP/Brevo credentials except this one (same centralization argument as
  the model-gateway).
- Email templates live here with the i18n catalogs; identity-service sends OTP/reset
  emails through events, keeping it a leaf service.
- At-least-once event delivery ⇒ notification creation must be idempotent (dedupe key in
  the event payload).
- Admin routes return **404 to non-admins** (no existence leakage) — preserved from the
  monolith's ADR-015, enforced via the platform-admin claim at the gateway + here.
- Impersonation *sessions* live in identity-service; the read-only guard for impersonated
  requests is enforced at the gateway. This service only surfaces the admin UI data.
- The four modules share the database but not tables; if one ever needs independent
  scaling (unlikely — all are low-traffic), the module boundary is the extraction seam.
- **Not carried over**: Dev Studio (stub analyzer — see
  [04 §4](../../04-functional-coverage.md)).

## Implementation checklist

- [ ] Schema + notification CRUD + read-state + preferences
- [ ] Email provider port + Brevo and log adapters
- [ ] Template rendering with bg/en catalogs
- [ ] `notification.requested` consumer (idempotent) + send job with retry/backoff
- [ ] Brevo webhook → delivery status
- [ ] Telegram ops-alert adapter
- [ ] Domain-event consumers (`invoice.issued`, `stock.low`)
- [ ] Support ticket model + tenant/staff routes
- [ ] `audit.event` consumer + queryable audit API (filters: tenant, user, action, period)
- [ ] Retention job with per-category policies
- [ ] Platform settings CRUD
- [ ] 404-for-non-admin behavior tests

## References

- [02 — service catalog entry](../../02-service-catalog.md#platform-service-8090)
- [02 § Deliberately merged boundaries](../../02-service-catalog.md#deliberately-merged-boundaries)
- [07 §5.11 — dependency graph](../../07-dependency-graphs.md#511-platform-service)
