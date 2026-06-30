# Nango

> The **OAuth / token backend** behind `integration-service`: it runs the OAuth dance,
> stores third-party SaaS tokens, and refreshes them — so our adapters call providers
> (Gmail, Drive, …) *on a user's behalf* without ever handling a token. We wrap it; we
> don't expose it. Non-OAuth connections (IMAP/SMTP, WebDAV basic auth) stay in our local
> vault.

**Type:** third-party engine (self-hosted, internal) · **Owner service:**
[integration-service](../../integration-service/README.md) (8080) · **Internal endpoints:**
`nango:3003` (proxy/API), `nango:3009` (Connect UI) · **Public:** no

## What it is

[Nango](https://nango.dev/) is an open-source **integration platform** for *outbound* API
connectivity. Its two core primitives are **Auth** (connect a user's external account —
OAuth2, API key, basic, JWT — with token storage + automatic refresh) and a **Proxy** that
makes authenticated calls to the provider by `connection_id`, injecting the valid token for
you. It covers 800+ APIs and ships two surfaces: an **admin dashboard** (register providers,
hold each provider's OAuth client credentials + scopes, inspect every connection and
token-refresh log) and a drop-in, white-label **Connect UI** widget that runs the end-user
consent + callback.

> **Not Authentik.** Authentik ([09 §3.1](../../../09-industry4z-platform-integration.md#31-identity--authentik-))
> is *inbound* auth — who logs **into** our platform. Nango is *outbound* auth — how we call
> a user's external SaaS account **on their behalf**. They don't overlap; adopting one does
> not remove the other.

## Why we use it

`integration-service` planned a hand-rolled OAuth flow + encrypted-credentials vault +
per-provider token-refresh logic. That is exactly what Nango does as a product — and getting
token refresh, revocation, and provider quirks subtly wrong is a real security and
reliability risk. Delegating the OAuth/token plane to Nango shrinks `integration-service`,
removes a class of bugs, and gives us a long catalog of providers behind one port.

| Nango gives us (don't build) | integration-service still owns (we build) |
|---|---|
| OAuth dance + callback for 800+ providers | The integration **catalog** + per-tenant install/connect lifecycle |
| **Token storage + automatic refresh** (on our infra) | The folder-discovered **adapter contract** (`connect`/`read`/verbs) |
| Connect UI (white-label connect-account widget) | Admin **access modes** (`all`/`excluded`/`exclusive`) |
| Provider-quirk handling, scopes, revocation | **Non-OAuth** creds (IMAP/SMTP, WebDAV) in the local vault |
| Per-user/per-tenant isolation via `connection_id` | Health surfacing, business rules, agent tool wiring |

## What we use it for

- **Agents acting on a user's behalf** — the `gmail_*`/`email_*`/drive tools resolve a Nango
  `connection_id`; the agent never sees a token (nothing sensitive enters the LLM context).
- **OAuth providers** (Google Workspace today; more for free as the catalog grows): connect,
  refresh, and proxied read/write calls.
- **Per-user / per-tenant isolation** — each user's Google account is a distinct
  `connection_id`, so "do this as *this* user" is just selecting the right connection.

## How it is wired in

- **Behind a port.** `integration-service`'s `google` adapter calls Nango through the
  service's OAuth port; Nango is the only thing that adapter knows about. The adapter
  contract and every caller-facing route are unchanged. `email`/`webdav` adapters don't
  touch Nango — they keep using the local AES-256-GCM vault (**split:** *OAuth → Nango,
  basic/secret → local vault*).
- **Connections reference, not store.** For OAuth providers, `integration.connections` keeps
  a `nango_connection_id` (and `auth_backend = 'nango'`) instead of a local `credentials`
  row; the token lives in Nango's Postgres on our infra. See
  [08 §4.8](../../../08-database-architecture.md#48-integration--connections--credential-vault).
- **Connect, then callback via webhook.** `POST /{provider}/connect` returns a Connect UI
  session token; the user finishes consent in the widget; Nango fires a `connection.created`
  webhook that `integration-service` records (replacing the hand-rolled `oauth/callback`).
- **On-demand reads/writes via Proxy.** Provider verbs go through Nango's proxy, which
  injects + refreshes the token. **Background syncs stay ours** — we already have Redis
  Streams + arq ([07 §7](../../../01-architecture-overview.md#7-asynchronous-work-and-events)),
  so scheduled polling lives in `integration-service`, not Nango.

## Deployment & access (two planes)

Neither surface is on the public internet (single-public-door invariant). Unlike LiteLLM /
Unstructured, Nango is **stateful** and ships as a small Compose stack (Server + Connect UI
+ its own **Postgres** + **Redis**), so treat it like a small stateful service — persistent
volume + backups for its Postgres, and `NANGO_ENCRYPTION_KEY` managed via **Infisical**.

- **Plane 1 — proxy/API + Connect UI (internal):** `nango:3003` (proxy/API) and `nango:3009`
  (Connect UI) on the private network. The only caller of the proxy is
  `integration-service`'s adapter; the Connect UI is reached by the browser through the
  gateway during a connect flow, never as a public ingress to Nango itself.
- **Plane 2 — admin dashboard (operators only):** not published publicly; reached over the
  **Tailscale** mesh VPN and additionally locked with Basic Auth (the dashboard is open to
  anyone who can reach the instance URL by default). Provider OAuth client secrets live here
  (sourced from Infisical), **not** in our DB.

## Edition note (free vs enterprise)

Self-hosted **free** Nango (docker-compose) covers exactly what this recommendation needs:
**OAuth + Connect UI + Proxy/token-refresh**, with credentials on our infrastructure.
Nango's managed **Syncs / Functions / Webhooks** and MCP server are **Enterprise-gated** — we
don't need them: background sync orchestration stays in our own arq jobs, and agents use our
existing tool layer, not Nango's MCP.

## Trade-off

Adds a **stateful** component (its own Postgres + Redis) and a small user-facing Connect UI —
a broader footprint than the purely request/response engines (LiteLLM, Unstructured). Accepted
because it removes security-sensitive OAuth/token code from `integration-service` and is
internal-only. If the near-term roadmap needs only IMAP/SMTP + WebDAV (non-OAuth) or a single
OAuth provider, Nango can be **deferred** with no architecture change — it lives behind the
adapter port either way. See
[09 §3.6](../../../09-industry4z-platform-integration.md#36-integrations--nango-).

## Value to the product & team

- **Product:** agents safely read/write a user's Gmail/Drive/Docs on their behalf; a
  ready-made connect-account UI; a long catalog of future SaaS connectors for low cost.
- **Team:** we stop maintaining the OAuth dance, token refresh, and per-provider quirks;
  credentials stay on our infra; `integration-service` stays focused on the catalog,
  lifecycle, access modes, and the adapter contract.

## References

- [integration-service README](../../integration-service/README.md) — the owning service.
- [09 §3.6 — Integrations: Nango](../../../09-industry4z-platform-integration.md#36-integrations--nango-)
- [09 §3.1 — Identity: Authentik](../../../09-industry4z-platform-integration.md#31-identity--authentik-) (why it's a different concern)
- [08 §4.8 — integration DB (connections & credential vault)](../../../08-database-architecture.md#48-integration--connections--credential-vault)
- [07 §5.10 — integration-service dependency graph](../../../07-dependency-graphs.md#510-integration-service)
