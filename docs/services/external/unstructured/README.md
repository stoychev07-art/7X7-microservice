# Unstructured

> The **document-parsing engine** behind `knowledge-service`: turns messy real-world files
> (PDF/DOCX/XLSX/HTML/images) into clean, structured, chunk-ready content. Internal-only and on
> the ingestion path only.

**Type:** third-party engine (self-hosted, internal) · **Owner service:**
[knowledge-service](../../knowledge-service/README.md) (8040) · **Internal endpoint:**
`unstructured:8000` · **Public:** no

## What it is

[Unstructured](https://unstructured.io/) is an open-source **document ingestion /
preprocessing engine** for the first mile of RAG. You hand it a raw file (PDF, DOCX, XLSX,
PPTX, HTML, EML, images, scanned docs) and it returns a normalized list of **document
elements** with types and metadata — titles, narrative text, list items, and **tables** —
plus OCR for image-only documents. Instead of one undifferentiated blob of text you get
semantically labeled blocks, which is what makes downstream chunking and retrieval good.

## Why we use it

Parsing real documents reliably is deceptively hard (reading order, multi-column layouts,
tables, encodings, scanned pages) and it is the single biggest lever on RAG quality — garbage
at the parse step means garbage chunks, embeddings, and answers. Rather than bundle and
maintain a stack of fragile parser libraries inside `knowledge-service`, we delegate the parse
step to Unstructured.

## What we use it for

- The **parse** step of the ingestion pipeline: `store original → parse (Unstructured) → chunk
  → embed → index (Qdrant)`.
- Broad format coverage and **OCR** for scanned/image documents (important for industrial docs:
  inspection reports, scanned forms).
- **Layout-aware chunking** — element types (titles, sections, tables) let the chunker split on
  semantic boundaries instead of blindly every N characters.

## How it is wired in

- **Behind a port.** `knowledge-service` calls Unstructured through a `DocumentParser` port;
  the Unstructured HTTP adapter is the only code that knows about it. Swapping the parser =
  one adapter.
- **Ingestion-path only.** It is called during upload/sync ingestion, **not** on the live
  `/search` query path — so it adds **no latency to agent retrieval**. If Unstructured is down,
  ingestion queues up but search keeps serving.
- **Internal only.** Reachable at `unstructured:8000` on the private network; never exposed
  publicly.

## Trade-off

Adopting it adds **one synchronous dependency** to `knowledge-service` at ingest time and a
new internal service to deploy/resource (layout + OCR models use real CPU/RAM). Accepted
because it is internal and ingestion-only — the blast radius is "uploads wait," not "users
can't get answers." See [09 §3.5](../../../09-industry4z-platform-integration.md#35-knowledge--qdrant--unstructured-).

## Value to the product & team

- **Product:** higher answer quality (cleaner chunks → better retrieval), broader format
  coverage, and OCR widen what is ingestible.
- **Team:** eliminates the maintenance burden of format parsers and their CVEs; parsing
  improvements arrive via a version bump; `knowledge-service` stays focused on its domain.

## References

- [knowledge-service README](../../knowledge-service/README.md) — the owning service.
- [09 §3.5 — Knowledge: Qdrant + Unstructured](../../../09-industry4z-platform-integration.md#35-knowledge--qdrant--unstructured-)
- [01 — key flow 3 (document ingestion)](../../../01-architecture-overview.md#key-flow-3--document-ingestion--retrieval)
- [07 §5.5 — knowledge-service dependency graph](../../../07-dependency-graphs.md#55-knowledge-service)
