# integration-service

> External-system connectivity behind a uniform, folder-discovered adapter contract:
> Google Workspace, IMAP/SMTP email, WebDAV, **e-signature** — plus the integration catalog and
> the encrypted credentials vault. OAuth providers delegate the OAuth dance + token refresh to
> **[Nango](../external/nango/README.md)** (self-hosted, behind a port); non-OAuth connections
> keep their secrets in the local vault; e-signature is delegated to
> **[Documenso](../external/documenso/README.md)** (self-hosted, behind a `SignaturePort`).

**Status:** 📋 planned · **Port (dev):** 8080 · **Database:** `integration`

## Responsibilities

- Integration **catalog** + per-tenant install/connect/disconnect lifecycle.
- **Credentials handling — split by connection type:**
  - **OAuth providers → [Nango](../external/nango/README.md)**: the OAuth dance and token
    storage/refresh are delegated to self-hosted Nango behind a port. We persist a
    `nango_connection_id` reference, not the token.
  - **Non-OAuth → local vault**: IMAP/SMTP and WebDAV basic-auth secrets are encrypted
    AES-256-GCM in the `credentials` table, never returned in plaintext over the API.
- **Adapters** (each a folder with `manifest` + adapter class, auto-discovered — the same
  plugin pattern as agents):
  - `google` — OAuth via **Nango** (workspace scopes; token refresh handled by Nango),
    Drive browse/upload/download, Gmail read/send/label/archive/filter (backs the `gmail_*`
    agent tools). Provider calls go through the Nango **proxy** by `connection_id` — no
    token ever enters our process or an agent's context.
  - `email` — universal IMAP/SMTP per-user connections, inbox/read/send/move + send log
    (backs the `email_*` agent tools).
  - `webdav` — connection CRUD, folder browse, file IO (consumed by knowledge-service's
    sync engine).
  - `esign` — **e-signature via [Documenso](../external/documenso/README.md)** (behind a
    `SignaturePort`): create a signing request from a generated document, track recipients +
    status, receive Documenso's completion webhook, and emit `document.signed`. Net-new
    capability — backs an `esign_*` agent tool / UI and the contract/invoice signing flow.
- Admin access modes per integration: `all` / `excluded` / `exclusive`.
- Connection health checks.

## API sketch

`GET /catalog` · `GET /connected` · `POST /{provider}/connect` · `DELETE /{provider}` ·
`POST /{provider}/oauth/callback` · `GET/POST /google/drive/**` · `POST /google/gmail/**` ·
`GET/POST /email/**` · `GET/POST /webdav/**` · `POST /esign/requests` · `GET /esign/requests/{id}` ·
`POST /esign/webhook` (Documenso `document.completed` sink)

For OAuth providers, `POST /{provider}/connect` returns a **Nango Connect UI session token**
(the frontend opens the widget) and the user finishes consent there; the
`POST /{provider}/oauth/callback` route becomes the sink for Nango's `connection.created`
**webhook**. `DELETE /{provider}` also revokes the connection in Nango. Non-OAuth providers
(`email`, `webdav`) keep the original connect/credentials flow against the local vault.

## Data owned

`connections` (with `auth_backend` + `nango_connection_id` for OAuth providers),
`credentials` (AES-256-GCM — **non-OAuth only**), `email_log`, `integration_access`,
`signature_requests` (e-signature lifecycle: `documenso_document_id`, recipients, status — the
signed PDF + certificate are retained by Documenso). OAuth tokens live in **Nango** (on our
infra), not in this DB — see
[08 §4.8](../../08-database-architecture.md#48-integration--connections--credential-vault).

## Dependencies

| Direction | What |
|---|---|
| Called by | agent-service (email/gmail/drive/esign tools), knowledge-service (sync file IO), gateway (UI) |
| Calls | **Nango** (OAuth/token backend + proxy, internal), **Documenso** (e-signature, internal — behind `SignaturePort`), Google APIs (via Nango proxy), IMAP/SMTP servers, WebDAV servers |
| Events | **out:** `document.signed` (→ platform-service notifications; optionally business-service). **in (optional trigger):** `invoice.issued` / contract events that should start a signing request |
| Jobs | connection health checks; scheduled syncs (our arq, **not** Nango) |

## Design notes

- The adapter base contract (`connect`, `disconnect`, `read`, `getStatus`, provider verbs)
  formalizes what the monolith's `integrations/registry.js` started — but here the *live*
  implementations (which the monolith kept in `core/`) move behind it too.
- **Nango is an implementation detail behind the OAuth port** — the adapter contract and all
  caller-facing routes are unchanged whether a provider's tokens are held by Nango or the
  local vault. Nango can be deferred (hand-roll one OAuth flow) with no contract change.
  Rationale + Authentik-vs-Nango distinction in
  [09 §3.6](../../09-industry4z-platform-integration.md#36-integrations--nango-).
- **Documenso is an implementation detail behind `SignaturePort`** — e-signature is a net-new
  capability (no existing port to swap), so it adds a `SignaturePort` + `document.signed` event
  rather than a swap. Signing lives here (an external-system integration), not in
  `document-service` (which only *produces* documents). The signed document is fetched from
  Documenso on demand; durable archival to our object storage is an optional follow-up.
  Documenso is **stateful** (own Postgres) and can be deferred with no contract change.
  Rationale in [09 §3.7](../../09-industry4z-platform-integration.md#37-documents--carbone--documenso-).
- Bundled verticals (Virtual Office, Energy — HMAC handshake) port as two more adapters
  **only if the partnerships are still live** ([04 §5](../../04-functional-coverage.md)).
- Microsoft / generic-ERP stubs from the monolith are **not** ported; the catalog entry
  returns when a real adapter exists.

## Implementation checklist

- [ ] Adapter protocol + folder discovery + manifest validation
- [ ] OAuth port + **Nango** adapter (Connect session, `connection.created` webhook sink,
      proxy calls by `connection_id`, disconnect→revoke); self-host Nango (Compose + Postgres
      + Redis + `NANGO_ENCRYPTION_KEY` via Infisical)
- [ ] Local credentials vault (AES-256-GCM, key rotation plan) — **non-OAuth only**
- [ ] Google adapter: OAuth via Nango, Drive IO, Gmail verbs (proxied)
- [ ] Email adapter: IMAP/SMTP connect-test, inbox/read/send, send log
- [ ] WebDAV adapter: connection CRUD, browse, streaming file IO
- [ ] **`esign` adapter + `SignaturePort`** (create request, recipients/status,
      `document.completed` webhook sink → `document.signed` event); `signature_requests` table;
      self-host Documenso (Compose + Postgres + SMTP + signing keys via Infisical)
- [ ] Catalog + install lifecycle + admin access modes
- [ ] Health-check job + status surfacing

## References

- [02 — service catalog entry](../../02-service-catalog.md#integration-service-8080)
- [Nango — OAuth/token backend (external engine)](../external/nango/README.md)
- [Documenso — e-signature engine (external)](../external/documenso/README.md)
- [09 §3.6 — Integrations: Nango](../../09-industry4z-platform-integration.md#36-integrations--nango-)
- [09 §3.7 — Documents: Carbone (+ Documenso)](../../09-industry4z-platform-integration.md#37-documents--carbone--documenso-)
- [06 §4.2 — plugin discovery pattern](../../06-architectural-patterns.md)
- [07 §5.10 — dependency graph](../../07-dependency-graphs.md#510-integration-service)
