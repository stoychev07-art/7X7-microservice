# manufacturing-service

> The production brain of the ERP: production orders, BOMs, operation routings, **MRP**,
> capacity planning, and shop-floor execution from **both** segments — automatic machine
> data (via SCADA) and manual confirmations (via MES terminals). It owns the *production
> plan and execution facts* and **references** stock, items, suppliers, telemetry, and
> documents from the services that already own them — it never duplicates them.

**Status:** 📋 planned (net-new vertical — no monolith counterpart; the SCADA+ERP
manufacturing core) · **Port (dev):** 8120 · **Database:** `manufacturing`

This service is the home for ТЗ **§4.1 (Производство и MRP)**, the **§3.2 MES terminal**
flow, and the order-linked half of the **§5 SCADA ↔ ERP integration**. Pure machine
telemetry (status, energy, alarms, historian — §3.1/§3.3) stays in
[iot-service](../iot-service/README.md); manufacturing-service correlates it to orders.

## Responsibilities

### Production orders & operations (§4.1)
- **Production orders** (производствени поръчки): number, finished item (БКТП/табло),
  quantity, due date, BOM + routing snapshot, source sales order (make-to-order), priority,
  status, derived `% complete`.
- **Operation routing** exploded per order (рязане → огъване → сглобяване → окабеляване →
  тест → експедиция): each operation bound to a work center, with planned vs actual times,
  good/scrap counts, and a per-operation state machine.
- **Scrap & rework** reporting per operation (with reason codes).

### MRP (§4.1 / §4.4)
- Level-by-level **BOM explosion** of open orders into time-phased material requirements.
- **Net requirement** = gross requirement − available stock − scheduled receipts + safety
  stock, computed against live stock read from [business-service](../business-service/README.md).
- Outputs **planned orders**: purchase requisitions (→ `purchase.requisition.requested`
  event for business-service purchasing) and planned production orders; **shortage signals**
  (→ `material.shortage`).
- Run modes: **on-release** (single-order availability check), **regenerative** (nightly
  full recompute), **net-change** (event-driven re-plan of affected items only).

### Capacity planning (§4.1)
- Work centers (machine or manual station) with a capacity calendar + shifts.
- **CRP load report**: scheduled operation time per work center vs available capacity
  (infinite-capacity load/bottleneck visualization first; finite scheduling later).

### MES terminal handling (§3.2)
- Operator login by **QR/PIN** (resolved to an identity-service user); a station maps to a
  work center.
- Serves the **active order + operation + instructions + BOM + reserved materials** to the
  terminal; accepts **start / pause / confirm** with good qty, scrap, and QR-scanned
  material consumption.
- Each confirmation is an **append-only fact** — the source for progress, OEE, and audit.

### SCADA ↔ ERP integration (§5)
- **SCADA → ERP** (order-linked facts): produced quantities per order, operation
  start/stop, downtime **with reason**, scrap — posted to `POST /ingest/production` through
  the **gateway** (service account, idempotent, buffering + retransmit on the SCADA side),
  the same gateway-only rule as the [Node-RED](../external/node-red/README.md) edge.
- **ERP → SCADA/MES** (active context): on release/operation activation the active order,
  BOM, operator instructions and reserved materials are published as an **event** consumed
  by the SCADA display edge — manufacturing does not hold a direct sync dependency on SCADA.
- **Energy/OEE correlation is left to the analytics layer**, not computed here: manufacturing
  owns the order→machine→time-window binding (carried on its events); iot-service owns the
  energy series; the §4.2 OEE/energy-per-order dashboards join the two.

## API sketch

`GET/POST /production-orders` · `POST /production-orders/{id}/release|complete|hold` ·
`GET /production-orders/{id}/progress` · `GET/POST /boms` · `GET/POST /routings` ·
`GET/POST /work-centers` · `POST /mrp/run` · `GET /mrp/messages` ·
`GET /mrp/planned-orders` · `GET /capacity/load` ·
`GET /mes/stations/{id}/active` · `POST /mes/operations/{id}/start|pause|confirm` ·
`POST /mes/scrap` · `POST /ingest/production` (SCADA/Node-RED, service account)

## Data owned

Engineering: `work_centers`, `boms`, `bom_lines`, `routings`, `routing_operations`,
`capacity_calendars`, `shifts`.
Execution: `production_orders`, `production_order_operations`, `order_materials`,
`production_confirmations` (append-only), `downtime_events`.
Planning: `mrp_runs`, `planned_orders`, `mrp_messages`.
Reference: `scrap_reasons`, `downtime_reasons`, `order_sequences`.
All tenant-scoped (`company_id`, RLS). See
[08 §4.12](../../08-database-architecture.md#412-manufacturing--production-orders-bom-routing-mrp).

Cross-service IDs (`item_id`, `counterparty_id`, `iot_device_id`, `operator_user_id`,
`instruction_doc_id`) are **logical references, not FKs** — those records live in
business-service / registry-service / iot-service / identity-service / document-service.

## Dependencies

| Direction | What |
|---|---|
| Called by | agent-service (tools: `production_order_create`, `mrp_run`, `production_status`, `work_confirm`, …), gateway (planner UI, MES terminals, SCADA bridge ingest) |
| Calls | **business-service** (stock availability for MRP, reserve/consume/scrap on confirm, item data), **document-service** (work-card / traveler PDF on release) |
| Events out | `production_order.released`, `production_order.completed`, `operation.confirmed`, `production.progress`, `material.shortage`, `purchase.requisition.requested`, `downtime.recorded`, `audit.event` |
| Events in | `sales_order.created` (make-to-order → create production order), `stock.*` change (net-change MRP) |
| Jobs | regenerative MRP (nightly), net-change MRP (queue), CRP load recompute, progress rollup, overdue/at-risk order sweep |
| Infra | `manufacturing` Postgres, Redis (queues + events); **no object storage** (PDFs live in document-service) |

Sync fan-out is **2** (business-service, document-service) — the
[≤2 rule](../../07-dependency-graphs.md#6-dependency-rules-the-invariants-behind-the-graphs)
holds. Everything else (SCADA progress, purchase requisitions, ERP→SCADA push, energy
correlation) is **events or gateway ingest**, not sync calls.

## Design notes

- **Boundary with business-service.** Stock is one ledger, owned by business-service.
  Manufacturing keeps the *per-order material plan* (`order_materials` — required / reserved
  / consumed tallies) and **asks** business-service to reserve on release and consume on MES
  confirmation. It never writes stock directly. Purchasing (requisitions → POs → goods
  receipt → three-way match) is a **business-service** concern; MRP only *requests* a
  requisition via event.
- **Boundary with iot-service.** iot-service owns raw machine telemetry/energy/alarms and
  the historian. Manufacturing owns order-linked production facts. They meet only at the
  **analytics layer** (OEE, kWh-per-order), which joins manufacturing's order/window binding
  with iot's series — neither service reaches into the other's DB.
- **Two progress sources, one model.** A `production_confirmation` carries `source = mes |
  scada`; the manual segment posts via the MES API, the machine segment via
  `POST /ingest/production`. Both are idempotent on `(operation, ts, source, key)` so SCADA
  buffering/retransmit (§5.3) is safe.
- **Make-to-order.** A sales order (business-service, future) or a deal-pipeline registry
  emits `sales_order.created`; manufacturing consumes it to spawn a production order — it
  does not own sales.
- **Snapshots for stability.** A production order snapshots the `bom_id` / `routing_id`
  versions it was planned with, so later engineering edits don't mutate in-flight orders.

## Implementation checklist

- [ ] Schema + Alembic baseline (engineering, execution, planning tables) + RLS
- [ ] Production-order + operation state machines; `order_number` sequence (per tenant)
- [ ] BOM explosion + routing explosion onto an order on release
- [ ] Stock reservation (business-service) on release; consumption on MES confirm; non-negativity respected upstream
- [ ] MRP engine (regenerative / net-change / on-release) → planned orders + `mrp_messages`
- [ ] `material.shortage` + `purchase.requisition.requested` events (idempotent)
- [ ] MES API: QR/PIN login, active-operation serve, start/pause/confirm, scrap
- [ ] `POST /ingest/production` (service account, idempotent) for SCADA/Node-RED facts
- [ ] ERP→SCADA active-context publish via `production_order.released` / `operation.confirmed`
- [ ] CRP capacity load report; progress rollup job
- [ ] Work-card / traveler render via document-service
- [ ] Agent tools wired in agent-service (write tools → approval interrupt)

## References

- [02 — service catalog entry](../../02-service-catalog.md#manufacturing-service-8120)
- [08 §4.12 — `manufacturing` database](../../08-database-architecture.md#412-manufacturing--production-orders-bom-routing-mrp)
- [07 §5.13 — dependency graph](../../07-dependency-graphs.md#513-manufacturing-service)
- [04 §7 — new manufacturing vertical](../../04-functional-coverage.md#7-new-manufacturing-vertical-scada--mes--mrp)
- [iot-service](../iot-service/README.md) (machine telemetry / energy historian) ·
  [business-service](../business-service/README.md) (stock / items / purchasing)
