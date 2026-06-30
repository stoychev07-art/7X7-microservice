# LiteLLM

> The **provider backend** behind `model-gateway`: one unified API over many LLM providers
> (cloud and local), with routing, fallback, retries, budgets, and per-model cost. We wrap it;
> we don't expose it.

**Type:** third-party engine (self-hosted, internal) · **Owner service:**
[model-gateway](../../model-gateway/README.md) (8030) · **Internal endpoint:** `litellm:4000`
· **Public:** no

## What it is

[LiteLLM](https://www.litellm.ai/) is an open-source **LLM gateway/proxy**. It exposes a
single OpenAI-compatible API and translates calls to 100+ providers behind it — Anthropic
(Claude), OpenAI, local **Ollama**, and more. It adds the cross-provider plumbing you'd
otherwise rewrite per provider: model aliasing, streaming normalization, retries/timeouts,
provider **fallback**, and per-key/per-model **budgets + rate limits**, plus a per-model cost
table and an admin UI for spend/keys/models.

## Why we use it

The platform is **multi-provider** — Anthropic/Claude in the cloud **and** local Ollama, with
more expected. Rather than hand-roll an adapter, retry, fallback, budget, and cost layer for
every provider, `model-gateway` delegates that **provider plumbing to LiteLLM** behind a
swappable port, and keeps only the business layer LiteLLM does not model.

| LiteLLM gives us (don't build) | model-gateway still owns (we build) |
|---|---|
| Unified API over Anthropic, Ollama, OpenAI, … | The internal contract `/v1/complete`, `/v1/embed` |
| Streaming normalization across providers | SSE passthrough on our contract |
| Retries, timeouts, **provider fallback** (cloud↔local) | `token.usage` outbox = **billing source of truth** |
| Per-key/per-model budgets + rate limits | **Tenant attribution** (`company_id` / `user_id`) |
| Per-model cost calculation | **Per-tenant kill switch** (business rule) |
| Model aliasing (logical name → provider+model) | Auth/headers + admin surface |

Full rationale (the LiteLLM-vs-model-gateway overlap and its resolution) is in
[09 §3.2](../../../09-industry4z-platform-integration.md#32-llm-access--litellm-vs-model-gateway-).

## What we use it for

- Every LLM **completion** and **embedding** in the product, routed through
  `model-gateway → LiteLLM → provider`.
- Switching/adding a provider (e.g. routing a model to local Ollama) = LiteLLM config + a
  logical-model mapping, **invisible to every agent manifest and caller**.
- Provider fallback (cloud ↔ local) and per-model budgets/rate limits.

## How it is wired in

- **Behind a port.** `model-gateway`'s domain talks to an `LLM` / `Embedder` port; the
  **LiteLLM adapter** is the only code that imports LiteLLM. Callers send a *logical* model
  name; mapping to provider+model is config. The engine is an implementation detail — an MVP
  may even start with a direct Anthropic adapter and switch the port to LiteLLM when the second
  provider (Ollama) lands; the contract never moves.
- **Metering stays ours.** `model-gateway` emits a `token.usage` event (tenant/user/feature
  attribution) on every call — **our event is billing truth**, LiteLLM's cost log is only for
  reconciliation. Billing must not depend on an external tool's retention.
- **Kill switch first.** The global / per-tenant AI kill switch is checked **before** a call
  ever reaches LiteLLM.

## Deployment & access (two planes)

Neither surface is on the public internet (single-public-door invariant):

- **Plane 1 — proxy API (machine-to-machine, internal only):** `litellm:4000` on the private
  network. Its **only** caller is `model-gateway`'s LiteLLM adapter with the master/virtual
  key. The product gateway (`:8000`) does **not** route to it.
- **Plane 2 — admin UI (`/ui`, operators only):** not published publicly; operators reach it
  over a **Tailscale** mesh VPN. It is an internal ops console (spend/keys/models/budgets) —
  **not** customer-facing and **not** a billing source of record. LiteLLM's own "users/teams/
  keys" are its entities, **not** our `company_id`/`user_id` tenants.

The canonical deployment runbook (Tailscale setup, port-binding, tailnet ACLs) lives with the
owning service: [model-gateway → LiteLLM deployment & access](../../model-gateway/README.md#litellm-deployment--access--two-planes).

## Value to the product & team

- **Product:** multi-provider reach (cloud + local) and provider fallback with no per-provider
  code; cost/budget guardrails; provider switches that never ripple into agents or callers.
- **Team:** we don't maintain provider SDKs, retry/fallback logic, or a cost table — that
  arrives via LiteLLM upgrades. Our codebase stays focused on tenancy, metering, and the kill
  switch.

## References

- [model-gateway README](../../model-gateway/README.md) — the owning service (contract,
  metering, deployment runbook).
- [09 §3.2 — LLM access: LiteLLM vs model-gateway](../../../09-industry4z-platform-integration.md#32-llm-access--litellm-vs-model-gateway-)
- [07 §5.4 — model-gateway dependency graph](../../../07-dependency-graphs.md#54-model-gateway)
- [06 — architectural patterns (centralized model access)](../../../06-architectural-patterns.md)
