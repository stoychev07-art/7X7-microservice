# model-gateway

> The only service that talks to LLM providers — a **thin owner backed by LiteLLM**.
> Uniform completion/embedding API, tenant-attributed token metering as a structural side
> effect of every call, and the per-tenant kill switch. LiteLLM supplies the provider
> plumbing (routing, fallback, retries, budgets, cost) across cloud and local models.

**Status:** 📋 planned · **Port (dev):** 8030 · **Database:** `modelgw`

## Why LiteLLM (and why model-gateway still exists)

The platform is **multi-provider** — cloud models (Anthropic/Claude) **and local models
(Ollama)**, with more providers expected. Rather than hand-roll a provider adapter, retry,
fallback, budget, and cost-calculation layer per provider, `model-gateway` delegates that
**provider plumbing to LiteLLM** behind a swappable port. What stays ours is the **business
layer** LiteLLM does not model: the internal contract, tenant attribution, the `token.usage`
outbox (billing source of truth), and the per-tenant kill switch. See the full rationale in
[09 §3.2](../../09-industry4z-platform-integration.md#32-llm-access--litellm-vs-model-gateway-).

| LiteLLM gives us (don't build) | model-gateway still owns (we build) |
|---|---|
| Unified API over Anthropic, Ollama, OpenAI, … | The internal contract `/v1/complete`, `/v1/embed` |
| Streaming normalization across providers | SSE passthrough on our contract |
| Retries, timeouts, **provider fallback** (cloud↔local) | `token.usage` outbox = **billing source of truth** |
| Per-key/per-model budgets + rate limits | **Tenant attribution** (our `company_id`/`user_id`) |
| Per-model cost calculation | **Per-tenant kill switch** (business rule) |
| Model aliasing (logical name → provider+model) | Auth/headers + admin surface |

> **Swappable backend.** LiteLLM sits behind the provider port, not baked into the domain.
> Sequencing: an MVP may start with a direct Anthropic adapter, then switch the port to
> LiteLLM the moment the **second provider (Ollama)** lands — the contract never moves.

## Responsibilities

- `POST /v1/complete` — chat completions with tool specs, streaming supported.
- `POST /v1/embed` — embeddings.
- Provider backend via **LiteLLM** (multi-provider: Anthropic/Claude cloud + local Ollama,
  more by config); credentials/keys encrypted AES-256-GCM. Adding a provider = LiteLLM
  config, not a new hand-written adapter.
- **Token metering**: every call emits a `token.usage` event with tenant/user/feature/agent
  attribution (passed as call metadata by the caller). Our event is the billing truth;
  LiteLLM's cost log is only for reconciliation.
- Global + per-tenant AI kill switch; provider balance/budget monitoring (daily check + alert).
- Retries, timeouts, fallback, and provider error normalization (inherited from LiteLLM).

## API sketch

`POST /v1/complete` (supports `stream=true`) · `POST /v1/embed` · `GET /v1/models` ·
`GET/POST /v1/providers` (admin) · `POST /v1/kill-switch` (admin)

## Data owned

`providers` / `model_configs` (logical-model → provider mapping, encrypted keys),
kill-switch state. Provider routing/budgets live in LiteLLM's config; we keep only what
billing and tenancy need.

## Dependencies

| Direction | What |
|---|---|
| Called by | agent-service (chat), knowledge-service (embeddings), document-service (AI import/format) |
| Calls | **LiteLLM** (provider backend → Anthropic/Claude, local Ollama, …) |
| Events out | `token.usage` (every call), `audit.event` |
| Jobs | provider balance/budget check (daily) |

## Design notes

- **Billing cannot be bypassed**: metering happens here, not at callsites — the structural
  fix for the monolith's per-callsite bookkeeping.
- Streaming passthrough must add near-zero latency; this sits on the hottest path.
- **`token.usage` must survive infrastructure failures** — it is billing data. Emission
  goes through a transactional outbox (usage row + pending event committed in one DB
  transaction, worker flushes to the stream), so a Redis outage can delay metering but
  never lose it (see [01 §7](../../01-architecture-overview.md#7-asynchronous-work-and-events)).
- Callers send a logical model name; mapping to provider+model is config (here + LiteLLM),
  so a provider switch — including routing to local Ollama — is invisible to every agent
  manifest.
- **LiteLLM is an implementation detail behind the port.** The domain never imports LiteLLM;
  swapping it (or starting with a direct adapter) leaves `/v1/complete`, `/v1/embed`, and
  `token.usage` untouched.

## LiteLLM deployment & access — two planes

LiteLLM exposes two surfaces that get two different access models. **Neither is published on
the public internet**, which preserves the single-public-door invariant (only the API gateway
is public).

- **Plane 1 — the proxy API (machine-to-machine, internal only).** LiteLLM runs as an
  internal service on the private network (e.g. `litellm:4000`), like every other core
  service. Its **only** caller is this service's LiteLLM adapter, over the internal network
  with the master/virtual key. The **product gateway (`:8000`) does not route to it**; end
  users and tenants reach LLMs solely through `agent-service → model-gateway`. LiteLLM is
  invisible to the product.
- **Plane 2 — the admin UI (`/ui`, for operators).** **Not exposed publicly** (no subdomain,
  no reverse-proxy ingress, no route on the product gateway). Operators reach it over a
  **private mesh VPN (Tailscale)** — the host joins the tailnet and the UI is reachable as if
  on the LAN. Secrets (`LITELLM_MASTER_KEY`, `UI_USERNAME`/`UI_PASSWORD`) come from the secret
  manager. See [Accessing the admin UI via Tailscale](#accessing-the-admin-ui-via-tailscale).

> **Boundary:** the LiteLLM admin UI is an **internal ops console** (spend/keys/models/budgets
> debugging) — **not** customer-facing and **not** a billing source of record. Customer-facing
> usage/billing stays in the product UI via `billing-service`. LiteLLM's "users/teams/keys"
> are its own entities — they are **not** our `company_id`/`user_id` tenants.

### Accessing the admin UI via Tailscale

The team reaches the LiteLLM UI through a **Tailscale** mesh VPN — no public exposure, no
per-session SSH tunnel, and it works for the whole ops team.

**One-time setup**

1. **Install Tailscale on the host** and bring it up (authenticate to your tailnet):

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up        # opens an auth URL; approve the machine in the admin console
```

2. **Keep the LiteLLM port off the public internet.** Either do **not** publish it to the host
   at all (reach it via the host's tailnet IP on the Docker network), or bind it to the host
   loopback / tailnet interface only — never `0.0.0.0`:

```yaml
services:
  litellm:
    # internal network for model-gateway → litellm; do NOT add a public ingress
    expose:
      - "4000"
    # optional: publish bound to loopback so the host (and thus tailnet) can reach it
    ports:
      - "127.0.0.1:4000:4000"
```

3. **Install Tailscale on each operator's laptop** and sign in to the same tailnet.

**Day-to-day access**

Open the UI using the host's Tailscale name (MagicDNS) or its tailnet IP:

```
http://<tailscale-host-name>:4000/ui
# or
http://100.x.y.z:4000/ui      # the host's 100.x tailnet IP
```

Then log in with the LiteLLM UI credentials (`UI_USERNAME` / `UI_PASSWORD`).

**Locking it down (recommended)**

- Restrict who can reach the UI with a **tailnet ACL** — only an `ops`/`admins` group may hit
  the LiteLLM host on port `4000`.
- Prefer **MagicDNS** names over raw IPs so ACLs and bookmarks stay stable.
- WireGuard is a valid substitute if you'd rather self-host the mesh; the access pattern is
  identical (host on the VPN, reach `:4000/ui` over the private interface).

## Implementation checklist

- [ ] Provider port (`LLM`, `Embedder` Protocols) + **LiteLLM adapter** (Anthropic/Claude + Ollama)
- [ ] Unified request/response DTOs incl. tool calls and usage block
- [ ] SSE/streaming passthrough
- [ ] `token.usage` event emission via transactional outbox (metering must survive Redis loss)
- [ ] Provider/model registry CRUD (admin) + AES-256-GCM credential storage
- [ ] Kill switch (global / per-tenant) checked on every call **before** hitting LiteLLM
- [ ] Balance/budget check job + `notification.requested` alert
- [ ] LiteLLM proxy on internal network only (no public ingress); port bound to loopback/tailnet, never `0.0.0.0`
- [ ] Tailscale on host + operator laptops; tailnet ACL limiting `:4000` to the ops group; admin UI reached at `http://<tailscale-host>:4000/ui`

## References

- [external/litellm — the provider engine behind this service](../external/litellm/README.md)
- [02 — service catalog entry](../../02-service-catalog.md#model-gateway-8030)
- [06 §5.4 — centralized model access](../../06-architectural-patterns.md)
- [07 §5.4 — dependency graph](../../07-dependency-graphs.md#54-model-gateway)
