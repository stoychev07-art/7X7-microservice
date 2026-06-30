# registry-service

> The structured-data backbone: user-configurable registries (dynamic tables) with
> templates, audit, access control, and semantic columns agents can reason about.

**Status:** 📋 planned · **Port (dev):** 8050 · **Database:** `registry`

## Responsibilities

- Dynamic registries: user-defined columns, JSONB rows, full CRUD with optimistic locking.
- Per-registry access matrix; audit trail; row revisions; XLSX export. The matrix is the
  **fine-grained, data-level grant layer** that sits *below* the platform RBAC
  ([01 §6](../../01-architecture-overview.md#authorization-model--roles--permissions-rbac)):
  coarse capability is gated by catalog permissions (`registry.read`/`registry.write` via
  `require_permission`), then this matrix scopes *which* registries a role/user may touch.
- **Registry templates** (9 domains × core/standard/pro tiers): counterparties, offers,
  contracts, assets, employees, projects, purchase/sales invoices, tasks.
- System registries seeded on `tenant.created`: work pipeline (Работен регистър),
  invoices (Фактури — until it graduates to business-service), personal + office tasks.
- **Canonical column roles** (`client_name`, `eik`, `offer_number`, …) so agent tools
  resolve fields semantically across differently-named tenant registries.
- Clients (counterparties) convenience API; dashboard briefing aggregation.

## API sketch

`GET/POST /registries` · `GET/POST /registries/{id}/columns` ·
`GET/POST/PATCH /registries/{id}/rows` (+ query, export) · `GET/POST /registries/{id}/access` ·
`GET /templates` · `POST /templates/{slug}/install` · `GET /clients/search` ·
`GET /dashboard/briefing`

## Data owned

`registries`, `registry_columns`, `registry_rows` (JSONB), `registry_access`,
`registry_audit`, `row_revisions`, `registry_templates`, `template_columns`.

## Dependencies

| Direction | What |
|---|---|
| Called by | agent-service (registry tools), business-service (counterparty lookup), document-service (row data for documents), gateway (UI) |
| Calls | nothing (leaf service) |
| Events in | `tenant.created` → seed system registries |
| Jobs | none (synchronous domain) |

## Design notes

- **Boundary with business-service**: flexible/user-defined data lives here; data whose
  incorrectness is illegal or financially wrong (invoices, stock) lives in
  business-service. See [02 — boundary table](../../02-service-catalog.md#boundary-with-registry-service-important).
- The monolith's 2507-line `registries/routes.js` decomposes into routes / domain /
  repository layers here — behavior parity (locking, audit, canonical roles) is the
  acceptance bar, covered by porting its test cases.
- Tasks are **not** a separate module: they ship as system registry templates
  (see [04 §3](../../04-functional-coverage.md)).

## Implementation checklist

- [ ] Schema + Alembic; JSONB row values + typed column definitions
- [ ] Row CRUD with optimistic locking (version column) + audit writes
- [ ] Access matrix enforcement (per-registry grants) layered under `require_permission` (`registry.read`/`registry.write`)
- [ ] Canonical-role resolution helper (used by agent tools)
- [ ] Template catalog + install endpoint; system-registry seeding consumer
- [ ] XLSX export; row revisions + restore
- [ ] Dashboard briefing aggregation
- [ ] RLS policies

## References

- [02 — service catalog entry](../../02-service-catalog.md#registry-service-8050)
- [04 — functional coverage mapping](../../04-functional-coverage.md)
- [07 §5.6 — dependency graph](../../07-dependency-graphs.md#56-registry-service)
