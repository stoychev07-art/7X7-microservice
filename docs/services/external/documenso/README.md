# Documenso

> The **e-signature engine** behind `integration-service`: send a generated document for
> signing, collect legally-binding signatures, and get back a signed PDF + audit certificate.
> E-signing is a **net-new capability** — it exists nowhere else in the architecture. We wrap
> it behind a `SignaturePort`; we don't expose it.

**Type:** third-party engine (self-hosted, **stateful**) · **Owner service:**
[integration-service](../../integration-service/README.md) (8080) · **Internal endpoint:**
`documenso:3000` · **Public:** no (signer pages reached through the gateway during a flow)

## What it is

[Documenso](https://documenso.com/) is the open-source **DocuSign alternative**. You upload a
PDF, place signature / date / text fields, assign recipients, and send a signing request;
signers complete it via an emailed link, and Documenso returns the **signed PDF plus a signing
certificate / audit trail**. It exposes an API and fires **webhooks** (e.g.
`document.completed`) so the calling system can react when signing finishes.

> **Why this is "new", not a swap.** Unlike Carbone/Qdrant/Unstructured (which replace
> something we already planned), e-signature has **no existing port** in
> [01](../../../01-architecture-overview.md)–[08](../../../08-database-architecture.md). Adopting
> Documenso therefore adds a new boundary — a `SignaturePort` and a new event — rather than
> swapping an adapter behind an existing one.

## Why we use it

Building e-signature in-house means handling cryptographic signing, tamper-evident audit
trails, multi-recipient routing, reminder emails, and legal-grade certificates — a deep domain
we have no reason to own. Documenso provides all of it as a self-hostable product, on our
infrastructure, behind one port.

| Documenso gives us (don't build) | integration-service still owns (we build) |
|---|---|
| Signing flow: fields, recipients, routing, reminders | The `SignaturePort` + `esign` adapter contract |
| Signed **PDF + audit certificate** | The **signature-request lifecycle** record (status, refs) |
| Signer-facing pages + signing emails | **Trigger policy** (which docs get signed, by whom) |
| Webhooks on completion/decline/expiry | Tenant scoping, event emission (`document.signed`) |
| Tamper-evidence / legal audit trail | Wiring to invoice/contract flows and agent tools |

## What we use it for

- **Contract & document signing** — a generated document (often rendered by
  [Carbone](../carbone/README.md) in `document-service`) is sent for signature to one or more
  recipients (typically the counterparty resolved via `registry-service` canonical roles).
- **Event- or API-triggered requests** — signing is started either by an explicit API call /
  agent tool, or by consuming a business event (e.g. an `invoice.issued` / contract flow that
  should produce a signature request). Signing is **not** auto-applied to every document.
- **Completion handling** — on the Documenso webhook, `integration-service` records the result
  and emits a `document.signed` event for notifications and downstream state.

## How it is wired in

- **Behind a port.** `integration-service` exposes a `SignaturePort`; the **`esign` adapter**
  (engine = Documenso) is the only code that talks to Documenso — the same folder-discovered
  adapter pattern as `google` / `email` / `webdav`. Swapping to another signing provider
  (e.g. DocuSign) is one adapter, no contract change.
- **Request → callback via webhook.** `POST /esign/requests` creates a signing request in
  Documenso and stores a local `signature_requests` row (status `sent`, `documenso_document_id`);
  when signing completes, Documenso fires a webhook that `integration-service` records and turns
  into a `document.signed` event.
- **Source documents.** The PDF to be signed is supplied by the caller (e.g. a
  `document-service` artifact URL). The completed signed PDF + certificate are retained by
  Documenso (on our infra) and fetched on demand through the adapter; durable archival into our
  own object storage is an optional follow-up if retention policy requires it.
- **No first-party fan-out.** The adapter calls only Documenso (external), so
  `integration-service` stays a leaf for internal synchronous traffic.

## Deployment & access (stateful)

Like [Nango](../nango/README.md) — and unlike the stateless engines (LiteLLM, Unstructured,
Carbone) — Documenso is **stateful** and ships as a small stack:

- **Internal only.** `documenso:3000` on the private network; the only caller is
  `integration-service`'s `esign` adapter. The signer-facing pages are reached by the browser
  **through the gateway** during a signing flow, never as a public ingress to Documenso itself.
- **Its own Postgres + SMTP.** Needs a persistent **Postgres** (signing data, audit trail) with
  volume + backups, and SMTP for signing emails. Secrets — DB URL, `NEXTAUTH_SECRET`, the
  signing certificate/encryption keys, SMTP creds — are managed via **Infisical**. Deployed by
  **Coolify**.
- **Admin surface** (operators) is not published publicly — reached over the **Tailscale** mesh
  VPN, same treatment as Nango's dashboard.

## Trade-off

Adds a **stateful** component (its own Postgres) plus a signer-facing surface and SMTP — a
broader footprint than request/response engines. Accepted because it delivers a whole capability
(legally-binding e-signature) we'd otherwise have to build, stays internal/behind a port, and
can be **deferred** with no architecture change (the `SignaturePort` exists either way). If
signing isn't in the near-term roadmap, skip it until a contract/invoice flow needs it.

## Value to the product & team

- **Product:** counterparties sign contracts/offers/invoices natively from generated documents;
  signed PDFs + audit certificates are first-class; the `invoice.issued`/contract flows can
  trigger a signature request via an event.
- **Team:** we don't build cryptographic signing, audit trails, or signer UX; signing stays an
  external-system integration behind one adapter, keeping `document-service` focused purely on
  *producing* documents.

## References

- [integration-service README](../../integration-service/README.md) — the owning service.
- [09 §3.7 — Documents: Carbone (+ Documenso)](../../../09-industry4z-platform-integration.md#37-documents--carbone--documenso-)
- [Carbone — the rendering engine that produces the documents to sign](../carbone/README.md)
- [02 — integration-service catalog entry](../../../02-service-catalog.md#integration-service-8080)
- [08 §4.8 — integration DB (connections, vault, signature requests)](../../../08-database-architecture.md#48-integration--connections--credential-vault)
- [07 §5.10 — integration-service dependency graph](../../../07-dependency-graphs.md#510-integration-service)
