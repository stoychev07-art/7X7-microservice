# document-service

> Turns business data into business documents: visual templates, PDF/Excel/Word
> generation, offers, plus the master price list, margins, and KSS that feed them. Rendering
> is delegated to **[Carbone](../external/carbone/README.md)** (one engine for PDF/DOCX/XLSX)
> behind a swappable render port.

**Status:** 📋 planned · **Port (dev):** 8060 · **Database:** `document`

## Responsibilities

- Visual document templates: section-based JSONB editor model.
- **Rendering via [Carbone](../external/carbone/README.md)** (behind a port): one engine does
  template → **PDF / DOCX / XLSX / ODS / HTML**, replacing the headless-Chromium PDF path *and*
  the separate Word/Excel libraries. Isolated here — render load never competes with other
  services.
- `generate_document` / `generate_excel` agent-tool backends produce native DOCX/XLSX via the
  same Carbone engine.
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
| Calls | model-gateway (AI import/format), registry-service (row data for document generation), **Carbone** (rendering, internal — behind the render port) |
| Jobs | price import processing |

> **Backed by [Carbone](../external/carbone/README.md)** behind the render port — one engine
> for PDF/DOCX/XLSX. Carbone is an internal-only implementation detail; the `POST /render` /
> `POST /generate` contract and all callers are unchanged whether rendering is done by Carbone
> or a fallback Chromium/library adapter.

## Design notes

- **Carbone is an implementation detail behind the render port.** The editable template source
  stays our section-JSONB model; the compiled Carbone template binary is stored in object
  storage and referenced from the template row. Carbone bundles a headless LibreOffice for PDF
  conversion, so it must run with hard memory/time limits (the same isolation the monolith's
  shared Puppeteer process lacked). Rationale + edition note in
  [external/carbone](../external/carbone/README.md) and
  [09 §3.7](../../09-industry4z-platform-integration.md#37-documents--carbone--documenso-).
- Pricing lives here (not in business-service) because its purpose is feeding offers/KSS;
  business-service reads prices over the API when building invoice lines.
- Artifacts are never written to local disk; stream to object storage and return a signed
  URL (the monolith's report agents already followed this rule).
- **Producing documents only — signing lives elsewhere.** E-signature (Documenso) is an
  external-system integration in `integration-service`, not a module here; the signed-document
  flow consumes documents this service produces. See
  [external/documenso](../external/documenso/README.md).

## Implementation checklist

- [ ] Template model + CRUD; section JSONB schema validation
- [ ] Render port + **Carbone adapter** (template + data → PDF/DOCX/XLSX → S3 + signed URL);
      compile section-JSONB → Carbone template, store binary in object storage; self-host
      Carbone (Compose, memory/time limits)
- [ ] xlsx/docx generation via the same Carbone engine
- [ ] Price list CRUD + history + XLSX import (AI-assisted via model-gateway)
- [ ] Margins with access checks
- [ ] KSS analyze/fill round-trip (port the monolith's Excel semantics — test with real KSS files)
- [ ] Offer draft endpoint for the agent tool

## References

- [02 — service catalog entry](../../02-service-catalog.md#document-service-8060)
- [07 §5.8 — dependency graph](../../07-dependency-graphs.md#58-document-service)
- [Carbone — rendering engine (external)](../external/carbone/README.md)
- [09 §3.7 — Documents: Carbone (+ Documenso)](../../09-industry4z-platform-integration.md#37-documents--carbone--documenso-)
