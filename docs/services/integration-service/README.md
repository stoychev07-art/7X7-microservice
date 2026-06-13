# integration-service

> External-system connectivity behind a uniform, folder-discovered adapter contract:
> Google Workspace, IMAP/SMTP email, WebDAV — plus the integration catalog and the
> encrypted credentials vault.

**Status:** 📋 planned · **Port (dev):** 8080 · **Database:** `integration`

## Responsibilities

- Integration **catalog** + per-tenant install/connect/disconnect lifecycle.
- **Credentials vault**: all external credentials encrypted AES-256-GCM, never returned in
  plaintext over the API.
- **Adapters** (each a folder with `manifest` + adapter class, auto-discovered — the same
  plugin pattern as agents):
  - `google` — OAuth (workspace scopes), Drive browse/upload/download, Gmail read/send/
    label/archive/filter (backs the `gmail_*` agent tools).
  - `email` — universal IMAP/SMTP per-user connections, inbox/read/send/move + send log
    (backs the `email_*` agent tools).
  - `webdav` — connection CRUD, folder browse, file IO (consumed by knowledge-service's
    sync engine).
- Admin access modes per integration: `all` / `excluded` / `exclusive`.
- Connection health checks.

## API sketch

`GET /catalog` · `GET /connected` · `POST /{provider}/connect` · `DELETE /{provider}` ·
`POST /{provider}/oauth/callback` · `GET/POST /google/drive/**` · `POST /google/gmail/**` ·
`GET/POST /email/**` · `GET/POST /webdav/**`

## Data owned

`connections`, `credentials` (encrypted), `email_log`, `integration_access`.

## Dependencies

| Direction | What |
|---|---|
| Called by | agent-service (email/gmail/drive tools), knowledge-service (sync file IO), gateway (UI) |
| Calls | Google APIs, IMAP/SMTP servers, WebDAV servers |
| Jobs | connection health checks |

## Design notes

- The adapter base contract (`connect`, `disconnect`, `read`, `getStatus`, provider verbs)
  formalizes what the monolith's `integrations/registry.js` started — but here the *live*
  implementations (which the monolith kept in `core/`) move behind it too.
- Bundled verticals (Virtual Office, Energy — HMAC handshake) port as two more adapters
  **only if the partnerships are still live** ([04 §5](../../04-functional-coverage.md)).
- Microsoft / generic-ERP stubs from the monolith are **not** ported; the catalog entry
  returns when a real adapter exists.

## Implementation checklist

- [ ] Adapter protocol + folder discovery + manifest validation
- [ ] Credentials vault (AES-256-GCM, key rotation plan)
- [ ] Google adapter: OAuth flow, Drive IO, Gmail verbs
- [ ] Email adapter: IMAP/SMTP connect-test, inbox/read/send, send log
- [ ] WebDAV adapter: connection CRUD, browse, streaming file IO
- [ ] Catalog + install lifecycle + admin access modes
- [ ] Health-check job + status surfacing

## References

- [02 — service catalog entry](../../02-service-catalog.md#integration-service-8080)
- [06 §4.2 — plugin discovery pattern](../../06-architectural-patterns.md)
- [07 §5.10 — dependency graph](../../07-dependency-graphs.md#510-integration-service)
