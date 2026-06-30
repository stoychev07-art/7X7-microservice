# document-service

> Turns business data into business documents: visual templates, PDF/Excel/Word
> generation, offers, plus the master price list, margins, and KSS that feed them.

**Status:** 📋 planned · **Port (dev):** 8060 · **Database:** `document`

## Responsibilities

- Visual document templates: section-based JSONB editor model.
- PDF rendering via headless Chromium (isolated here — rendering load never competes with
  other services).
- Excel (`xlsx`) and Word (`docx`) generation; `generate_document` / `generate_excel`
  agent tools backend.
- Offer drafting (`offer_draft` tool backend).
- **Master price list**: categories, items, price history, AI-assisted XLSX import
  (via model-gateway).
- **Margins**: per category/item, access-controlled; `save_margins` tool backend.
- **KSS** (construction cost sheets): analyze + fill, Excel round-trip.

## API sketch

`GET/POST /templates` · `POST /render` (PDF) · `POST /generate` ·
`GET/POST /prices/categories` · `GET/POST /prices/items` · `POST /prices/import` ·
`GET/POST /margins` · `POST /kss/analyze` · `POST /kss/fill`

## Data owned

`document_templates`, `price_categories`, `price_items`, `price_history`,
`price_imports`, `category_margins`, `item_margins`, `margin_access`.
Generated artifacts in S3/MinIO with signed download URLs.

## Dependencies

| Direction | What |
|---|---|
| Called by | agent-service (document/price/KSS tools), business-service (invoice PDFs, price reads), gateway (UI) |
| Calls | model-gateway (AI import/format), registry-service (row data for document generation) |
| Jobs | price import processing |

## Design notes

- Chromium rendering should run in its own worker pool with hard memory/time limits — in
  the monolith Puppeteer shared the API process.
- Pricing lives here (not in business-service) because its purpose is feeding offers/KSS;
  business-service reads prices over the API when building invoice lines.
- Artifacts are never written to local disk; stream to object storage and return a signed
  URL (the monolith's report agents already followed this rule).

## Implementation checklist

- [ ] Template model + CRUD; section JSONB schema validation
- [ ] Render pipeline: template + data → HTML → Chromium PDF → S3 + signed URL
- [ ] xlsx/docx generation helpers
- [ ] Price list CRUD + history + XLSX import (AI-assisted via model-gateway)
- [ ] Margins with access checks
- [ ] KSS analyze/fill round-trip (port the monolith's Excel semantics — test with real KSS files)
- [ ] Offer draft endpoint for the agent tool

## References

- [02 — service catalog entry](../../02-service-catalog.md#document-service-8060)
- [07 §5.8 — dependency graph](../../07-dependency-graphs.md#58-document-service)
