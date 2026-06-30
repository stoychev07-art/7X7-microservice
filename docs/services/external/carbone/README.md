# Carbone

> The **document-rendering engine** behind `document-service`: one engine that turns a
> template + JSON data into **PDF / DOCX / XLSX / ODS / HTML**. It replaces the planned
> headless-Chromium PDF path *and* the separate Word/Excel generation libraries with a single
> template model. We wrap it; we don't expose it.

**Type:** third-party engine (self-hosted, internal) · **Owner service:**
[document-service](../../document-service/README.md) (8060) · **Internal endpoint:**
`carbone:4000` · **Public:** no

## What it is

[Carbone](https://carbone.io/) is an open-source **report/document generator**. You design a
template in a normal editor (LibreOffice / Word / Excel → DOCX, XLSX, ODT, ODS, HTML) and mark
it up with `{d.field}` placeholders, loops (`{d.items[i].name}`), and formatters; you then POST
the template + a JSON payload and Carbone renders the final document. PDF output is produced by
a headless **LibreOffice** that Carbone manages internally, so a single engine covers both the
"pretty PDF" and the "native Office file" cases.

## Why we use it

`document-service` planned **two** rendering mechanisms: PDF via headless Chromium (Puppeteer)
and Word/Excel via separate generation libraries. That is two stacks to run, secure, and
resource-limit. Carbone collapses them into **one template→document engine**, and its
"design a visual template, fill placeholders" model is a direct match for our existing
section-based **visual templates** concept — so adopting it shrinks the rendering surface
without changing what callers see.

| Carbone gives us (don't build) | document-service still owns (we build) |
|---|---|
| Template → **PDF / DOCX / XLSX / ODS / HTML** in one engine | The render contract `POST /render`, `POST /generate` |
| Headless-LibreOffice PDF conversion (no separate Chromium pool) | Visual **template model** (section JSONB editor) + versioning |
| Loops, conditions, number/date/currency formatters | Tenant scoping, access control, the **price list / margins / KSS** domains |
| Native Office output (real `.docx`/`.xlsx`, not HTML hacks) | Artifact persistence to object storage + **signed download URLs** |

## What we use it for

- **PDF rendering** (`POST /render`) — invoices, offers, and template-based documents.
- **Office generation** (`POST /generate`) — `generate_document` / `generate_excel` agent-tool
  backends and KSS Excel round-trips, as real DOCX/XLSX.
- **Offer drafting** and any template that previously needed Chromium *or* a doc/xlsx library
  — now a single code path.

## How it is wired in

- **Behind a port.** `document-service`'s domain talks to a render-engine port (`PdfEngine` /
  render adapter); the **Carbone adapter** is the only code that knows about Carbone. The
  service API (`POST /render`, `POST /generate`) and every caller — `agent-service` (document
  tools), `business-service` (invoice/offer PDFs), the UI via the gateway — are unchanged.
- **Templates compile from our model.** The editable source stays our section-JSONB
  `document_templates`; the compiled Carbone template binary is stored in object storage and
  referenced by the template row (see [08 §4.6](../../../08-database-architecture.md#46-document--templates-price-list-margins)).
- **Artifacts, not local disk.** Rendered files stream straight to S3/MinIO (the
  S3-compatible object-storage port) and the API returns a **signed URL** — the existing
  "never write to local disk" rule is preserved.
- **Resource isolation.** Carbone's LibreOffice workers are the new home of the "render load
  never competes with other services" design note — run it with hard memory/time limits, the
  same constraint previously placed on the Chromium pool.

## Deployment & access

Stateless request/response engine — like [LiteLLM](../litellm/README.md) and
[Unstructured](../unstructured/README.md), **not** stateful like [Nango](../nango/README.md):

- **Internal only.** Reachable at `carbone:4000` on the private network; the **only** caller is
  `document-service`'s render adapter. Never published publicly (single-public-door invariant —
  only the gateway `:8000` is exposed).
- **No database, no persistent state.** Templates + data come in on the request, the rendered
  file goes out. A scratch/tmp volume plus CPU/RAM limits are advisable because LibreOffice is
  heavy. Deployed by **Coolify**; any license/config via **Infisical**.

## Edition note (free vs enterprise)

Carbone's rendering core is open-source and self-hostable, which covers exactly what we need:
**template → PDF/DOCX/XLSX** behind our port. Carbone's hosted/Studio conveniences are not
required — template authoring uses our own editor and the compiled template lives in our object
storage. If the on-prem rendering daemon's edition ever constrains us, the port lets us swap the
adapter (even back to a Chromium/library path) with no contract change.

## Trade-off

Adds **one internal engine** (with a bundled LibreOffice) to deploy and resource. Accepted
because it *removes* two existing mechanisms (Chromium pool + doc/xlsx libraries), is
internal-only, and sits behind a port. LibreOffice rendering can be CPU/RAM-spiky — isolate it
and cap it, exactly as the Chromium pool was meant to be.

## Value to the product & team

- **Product:** consistent, well-formatted PDFs **and** native Office files from one template
  model; the visual-template concept maps cleanly onto Carbone.
- **Team:** one engine instead of two rendering stacks to maintain and patch; rendering
  improvements arrive via a version bump; `document-service` stays focused on templates,
  pricing, margins, and KSS.

## References

- [document-service README](../../document-service/README.md) — the owning service.
- [09 §3.7 — Documents: Carbone (+ Documenso)](../../../09-industry4z-platform-integration.md#37-documents--carbone--documenso-)
- [02 — document-service catalog entry](../../../02-service-catalog.md#document-service-8060)
- [07 §5.8 — document-service dependency graph](../../../07-dependency-graphs.md#58-document-service)
- [08 §4.6 — document DB (templates, price list, margins)](../../../08-database-architecture.md#46-document--templates-price-list-margins)
