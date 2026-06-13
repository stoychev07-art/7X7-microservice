# business-service

> First-class ERP domain logic: invoicing, inventory, and spendings — the entities whose
> invariants must be enforced by code, not by the flexible registry engine.

**Status:** 📋 planned (net-new — no monolith counterpart, built post-parity) ·
**Port (dev):** 8100 ·
**Database:** `business`

## Responsibilities

### Invoicing
- Sales & purchase invoices as typed entities with line items.
- **Sequential legal numbering per tenant** (Bulgarian requirement) via dedicated
  sequence rows — no gaps, no duplicates, safe under concurrency.
- VAT calculation; credit/debit notes; statuses (draft → issued → paid/overdue/void).
- **Immutability after issue** — corrections only via credit/debit notes.
- PDF rendering delegated to document-service.

### Inventory
- Items/SKUs, warehouses.
- Stock movements (receipt, issue, transfer, adjustment) as an **append-only ledger**;
  stock levels derived/materialized, never edited directly; stock cannot go negative.
- Reservations; minimum-stock thresholds → `stock.low` events.

### Spendings
- Expense records with categories and supplier (counterparty) linkage.
- Recurring expenses; simple budgets; cash-flow summary reports.

## API sketch

`GET/POST /invoices` · `POST /invoices/{id}/issue` · `POST /invoices/{id}/void` ·
`POST /invoices/{id}/credit-note` · `GET/POST /items` · `GET/POST /warehouses` ·
`POST /stock/movements` · `GET /stock/levels` · `GET/POST /expenses` ·
`GET /reports/cashflow`

## Data owned

`invoices`, `invoice_lines`, `invoice_sequences`, `items`, `warehouses`,
`stock_movements`, `stock_levels` (materialized), `expenses`, `expense_categories`,
`budgets`.

## Dependencies

| Direction | What |
|---|---|
| Called by | agent-service (tools: `invoice_create`, `invoice_list`, `inventory_check`, `stock_move`, `expense_add`, `expense_summary`), gateway (UI) |
| Calls | registry-service (counterparty lookup by canonical role), document-service (PDF render, price list reads) |
| Events out | `invoice.issued`, `stock.low` (→ platform-service notifications) |
| Jobs | overdue-invoice sweep, low-stock check, recurring-expense generation |

## Design notes

- **Boundary rule**: if a wrong value is merely messy → registry-service; if it is illegal
  or financially incorrect → here. The deal-pipeline registry references invoices by ID,
  never duplicates them.
- **Migration sequencing**: during strangler parity the monolith's "Фактури" system
  registry ports unchanged; once this service ships, its rows graduate into typed invoices
  (canonical-role columns make the mapping mechanical) — see
  [04 §6](../../04-functional-coverage.md#6-new-capabilities-beyond-parity-business-service).
- Counterparties stay in registry-service (tenant-flexible); this service stores only
  their IDs + denormalized display snapshots on issued invoices (legal documents must not
  change when the counterparty record is edited).

## Implementation checklist

- [ ] Schema + Alembic; invoice sequence allocation (SELECT … FOR UPDATE per tenant/series)
- [ ] Idempotent write endpoints — dedupe on the caller's idempotency key (agent write
      tools send an approval-derived key); invoice numbering must never double-allocate
      on a replay
- [ ] Invoice lifecycle state machine + immutability enforcement + VAT math (tested hard)
- [ ] Credit/debit note flows
- [ ] Stock movement ledger + materialized levels + non-negativity constraint
- [ ] Expense CRUD + recurring generation job + cashflow report
- [ ] `invoice.issued` / `stock.low` events
- [ ] Agent tools wired in agent-service (write tools → approval interrupt)
- [ ] PDF rendering via document-service template

## References

- [02 — service catalog entry + boundary table](../../02-service-catalog.md#business-service-8100)
- [04 §6 — new capabilities beyond parity](../../04-functional-coverage.md#6-new-capabilities-beyond-parity-business-service)
- [07 §5.7 — dependency graph](../../07-dependency-graphs.md#57-business-service)
