# iot-service

> The IoT vertical: a device/sensor **registry**, **time-series storage** in
> **[TimescaleDB](../../08-database-architecture.md#411-iot--devices-readings-alert-rules-timescaledb)**,
> **per-organization alert rules**, and anomaly/alert **event emission**. Sensor data arrives
> already decoded from the **[Node-RED](../external/node-red/README.md)** ingestion edge (which
> speaks MQTT/Modbus/OPC UA and posts normalized readings through the gateway). It is the **only
> first-party owner** of the IoT plane and the **single trusted source** of the
> `device → company` mapping.

**Status:** 📋 planned · **Port (dev):** 8110 · **Database:** `iot` (**TimescaleDB** = Postgres + extension)

## Responsibilities

- **Device/sensor registry** — devices, sensors, location, model, and the **`device → company_id`
  mapping**. This mapping is the trusted tenancy source: every reading and every emitted event
  derives its `company_id` here, never from an upstream tool (Node-RED carries none).
- **Ingestion API** — accept normalized readings from Node-RED (`POST /readings`, batched),
  validate, and append to TimescaleDB. Idempotent on `(device_id, ts, metric)` so edge retries
  are safe.
- **Time-series storage** — `sensor_readings` as a TimescaleDB **hypertable**; **continuous
  aggregates** (e.g. 1-min/1-hour rollups) power dashboards cheaply; **retention** + compression
  policies drop/compact old raw data automatically.
- **Per-organization alerting (our own — no Grafana)** — tenant-defined `alert_rules` (metric,
  condition `> X for N min`, severity, target), tenant-scoped + RLS, managed via our API/UI and
  the `iot.alerts.manage` permission. A rule-evaluation worker checks recent windows and, on a
  breach, emits the alert event with the trusted `company_id`.
- **Dashboards backend** — query/aggregate endpoints consumed by **our Next.js app** (gateway +
  generated TS client + our RBAC). No external dashboard tool, no read-only DB role on `iot`.
- **Anomaly knowledge** — push anomaly descriptions/diagnoses as embeddings into
  knowledge-service's `iot_anomalies` namespace so `/agents/iot` can do semantic search over
  past incidents.

## API sketch

`GET/POST /devices` · `GET/POST /devices/{id}/sensors` ·
`POST /readings` (bulk ingest — Node-RED) ·
`GET /readings` (range query) · `GET /metrics/{device_id}` (continuous-aggregate series) ·
`GET/POST /alert-rules` · `PATCH /alert-rules/{id}` (tenant-managed, `iot.alerts.manage`) ·
`GET /alerts` (history) · `GET /dashboard/overview`

## Data owned

`devices`, `sensors`, `sensor_readings` (**hypertable**), continuous aggregates
(`readings_1m`, `readings_1h`), `alert_rules`, `alerts`. All tenant-scoped (`company_id`, RLS).
Large/binary payloads (camera frames, waveforms), **if any**, go to S3/MinIO with a key stored
on the reading — never inline in the DB.

## Dependencies

| Direction | What |
|---|---|
| Called by | **Node-RED** (ingestion, via gateway + service account), gateway (UI: dashboards, rule management), agent-service (`/agents/iot` reads device/series context) |
| Calls | model-gateway (anomaly explanation — via agent-service), knowledge-service (write anomaly embeddings to `iot_anomalies`) |
| Events out | `sensor.anomaly`, `device.alert` → platform-service (notify) and agent-service (diagnosis) — both carry the trigger-derived `company_id` |
| Jobs | alert-rule evaluation (windowed), continuous-aggregate refresh, retention/compression policy maintenance |
| Infra | **TimescaleDB** (`iot` DB), Redis (eval queue, events), optional S3/MinIO (binary payloads) |

> **Backed by [Node-RED](../external/node-red/README.md)** as the ingestion edge — it decodes
> MQTT/Modbus/OPC UA and forwards normalized readings **through the gateway only** (same
> gateway-only rule as n8n). Node-RED is an implementation detail of *how data arrives*; the
> `POST /readings` contract is unchanged whether readings come from Node-RED or a plain HTTP
> sensor.

## Design notes

- **TimescaleDB is still database-per-service.** It is PostgreSQL 16 + the Timescale extension,
  so `iot` is owned exclusively by this service with the same `asyncpg`, Alembic, and RLS as
  every other DB. The hypertable + continuous aggregates are an internal storage choice, not a
  new datastore technology. Rationale in
  [09 §3.10](../../09-industry4z-platform-integration.md#310-iot-vertical--iot-service--timescaledb--node-red--decided-adopt-grafana-rejected).
- **Grafana is deliberately not used.** Dashboards live in our Next.js app and alert rules are
  first-party tenant data — this keeps one auth model (our RBAC), structural tenant isolation
  (`company_id` from the gateway/registry, not a Grafana Org), and a private `iot` database.
  See the Grafana-rejection table in
  [09 §3.10](../../09-industry4z-platform-integration.md#grafana--rejected-we-build-our-own-multi-tenant-alerting--dashboards).
- **Tenancy is structural, not convention.** Node-RED and any other ingestion source send only
  `device_id`; this service resolves `device → company_id`. Emitted events therefore always
  carry a *trusted* `company_id` (the same rule that lets n8n run the diagnosis loop safely —
  [n8n tenant-context constraint](../external/n8n/README.md#tenant-context--billing-the-core-constraint)).
- **The alert loop.** rule breach → `iot-service` emits `sensor.anomaly`/`device.alert` →
  `platform-service` notifies the org, and (optionally) `n8n` runs *event → `/agents/iot`
  (Claude diagnosis via model-gateway) → notify*. The LLM call is attributed to the device's
  `company_id`, so `token.usage` bills the right org.
- **Binary payloads decide S3 scope.** Scalar readings (temp/pressure/counts) need only
  TimescaleDB; camera frames/vibration waveforms go to S3/MinIO (key on the reading row).

## Implementation checklist

- [ ] Device/sensor registry CRUD + `device → company_id` mapping (RLS)
- [ ] `POST /readings` bulk ingest (idempotent on `device_id, ts, metric`); Node-RED service account + `iot.readings.write` permission
- [ ] TimescaleDB hypertable + continuous aggregates (1m/1h) + retention/compression policies (Alembic baseline)
- [ ] Range-query + aggregate endpoints for the Next.js dashboards
- [ ] `alert_rules` CRUD (tenant-managed, `iot.alerts.manage`) + windowed evaluation worker
- [ ] Emit `sensor.anomaly` / `device.alert` (with trusted `company_id`) via the event/outbox
- [ ] Anomaly-embedding push to knowledge-service `iot_anomalies` namespace
- [ ] (If needed) S3/MinIO path for binary sensor payloads

## References

- [02 — service catalog entry](../../02-service-catalog.md#iot-service-8110)
- [08 §4.11 — `iot` database (TimescaleDB)](../../08-database-architecture.md#411-iot--devices-readings-alert-rules-timescaledb)
- [07 §5.12 — dependency graph](../../07-dependency-graphs.md#512-iot-service)
- [Node-RED — ingestion edge (external)](../external/node-red/README.md)
- [09 §3.10 — IoT vertical (decision: adopt; Grafana rejected)](../../09-industry4z-platform-integration.md#310-iot-vertical--iot-service--timescaledb--node-red--decided-adopt-grafana-rejected)
