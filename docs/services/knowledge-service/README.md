# knowledge-service

> Documents in, grounded answers out: library, parsing, chunking, embeddings, namespaced
> vector search, RAG facts, projects, and the file-sync engines.

**Status:** 📋 planned · **Port (dev):** 8040 · **Database:** `knowledge` (+ Qdrant vector store)

## Responsibilities

- Document library: categories, file upload (PDF/DOCX/XLSX), bulk upload.
- Parsing (via the internal **Unstructured** service) → chunking → embedding (via
  model-gateway) → Qdrant collection per **namespace**.
- `POST /search`: namespace-scoped hybrid retrieval (vector + keyword) — the backend of the
  `knowledge_search` agent tool and of manifest `data_namespaces`.
- RAG facts: promote curated content from documents into a facts corpus.
- Projects as retrieval scopes (active project per user, Redis).
- Archive views: sources, stats, reindex, permissions matrix.
- **Sync engines**: WebDAV and Google Drive folder sync — per-file transactions, 30-min
  sweeps, wall-clock deadline + continuation re-enqueue.

## API sketch

`POST /files` · `GET /files` · `GET/POST /categories` · `POST /search` ·
`POST /facts/promote` · `GET/POST /projects` · `GET/POST /sync/folders` · `POST /sync/run`

## Data owned

`doc_categories`, `doc_files`, `document_chunks` (vectors in Qdrant), `rag_facts`, `projects`,
`sync_folders`, `sync_state`. Originals in S3/MinIO.

## Dependencies

| Direction | What |
|---|---|
| Called by | agent-service (search/retrieve), gateway (library UI) |
| Calls | Unstructured (file parsing, internal), model-gateway (embeddings), integration-service (WebDAV/Drive credentials + file IO) |
| Events out | `document.ingested` |
| Jobs | embed-document, sync-webdav, sync-drive, sync sweeps (30 min), reindex |

## Design notes

- **Fixes the monolith's open tech-debt item by design**: Drive sync uses per-file commits
  (the monolith's WebDAV fix, applied universally) — no long transactions pinning locks.
- Namespaces are the contract with agent manifests; keep them cheap to create.
- The vector store is **Qdrant**, behind a `VectorStore` port — swappable to another engine
  with one adapter (see [03 §5](../../03-agent-platform.md#5-swapping-infrastructure)).
- File parsing is delegated to the internal **Unstructured** service (`unstructured:8000`),
  behind a `DocumentParser` port — internal-only and on the ingestion path only (not on the
  `/search` query path), so it adds no latency to agent retrieval.

## Implementation checklist

- [ ] Schema + Qdrant setup; namespace model
- [ ] `DocumentParser` port + Unstructured adapter (`unstructured:8000`)
- [ ] Upload → parse (Unstructured) → chunk → embed pipeline (arq)
- [ ] `POST /search` with namespace fan-out + score merge
- [ ] `document.ingested` event
- [ ] WebDAV sync engine (per-file txn, deadline + continuation)
- [ ] Drive sync engine (same pattern, via integration-service IO)
- [ ] Sweep jobs + singleton dedupe keys
- [ ] Facts promotion + projects + archive permission checks

## References

- [external/unstructured — the parsing engine behind this service](../external/unstructured/README.md)
- [01 — key flow 3 (ingestion diagram)](../../01-architecture-overview.md)
- [02 — service catalog entry](../../02-service-catalog.md#knowledge-service-8040)
- [07 §5.5 — dependency graph](../../07-dependency-graphs.md#55-knowledge-service)
