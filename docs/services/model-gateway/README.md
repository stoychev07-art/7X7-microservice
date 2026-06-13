# model-gateway

> The only service that talks to LLM providers. Uniform completion/embedding API,
> provider switching, and token metering as a structural side effect of every call.

**Status:** 📋 planned · **Port (dev):** 8030 · **Database:** `modelgw`

## Responsibilities

- `POST /v1/complete` — chat completions with tool specs, streaming supported.
- `POST /v1/embed` — embeddings (OpenAI initially).
- Provider registry (admin-managed): Anthropic (primary chat), OpenAI (embeddings);
  credentials encrypted AES-256-GCM. Adding a provider = one new adapter.
- **Token metering**: every call emits a `token.usage` event with tenant/user/feature/agent
  attribution (passed as call metadata by the caller).
- Global + per-tenant AI kill switch; provider balance monitoring (daily check + alert).
- Retries, timeouts, and provider error normalization.

## API sketch

`POST /v1/complete` (supports `stream=true`) · `POST /v1/embed` · `GET /v1/models` ·
`GET/POST /v1/providers` (admin) · `POST /v1/kill-switch` (admin)

## Data owned

`providers` (encrypted credentials), `model_configs`, kill-switch state.

## Dependencies

| Direction | What |
|---|---|
| Called by | agent-service (chat), knowledge-service (embeddings), document-service (AI import/format) |
| Calls | Anthropic API, OpenAI API |
| Events out | `token.usage` (every call), `audit.event` |
| Jobs | provider balance check (daily) |

## Design notes

- **Billing cannot be bypassed**: metering happens here, not at callsites — the structural
  fix for the monolith's per-callsite bookkeeping.
- Streaming passthrough must add near-zero latency; this sits on the hottest path.
- **`token.usage` must survive infrastructure failures** — it is billing data. Emission
  goes through a transactional outbox (usage row + pending event committed in one DB
  transaction, worker flushes to the stream), so a Redis outage can delay metering but
  never lose it (see [01 §7](../../01-architecture-overview.md#7-asynchronous-work-and-events)).
- Callers send a logical model name; mapping to provider+model is config here, so a
  provider switch is invisible to every agent manifest.

## Implementation checklist

- [ ] Provider port (`LLM`, `Embedder` Protocols) + Anthropic and OpenAI adapters
- [ ] Unified request/response DTOs incl. tool calls and usage block
- [ ] SSE/streaming passthrough
- [ ] `token.usage` event emission via transactional outbox (metering must survive Redis loss)
- [ ] Provider registry CRUD (admin) + AES-256-GCM credential storage
- [ ] Kill switch (global / per-tenant) checked on every call
- [ ] Balance check job + `notification.requested` alert

## References

- [02 — service catalog entry](../../02-service-catalog.md#model-gateway-8030)
- [06 §5.4 — centralized model access](../../06-architectural-patterns.md)
- [07 §5.4 — dependency graph](../../07-dependency-graphs.md#54-model-gateway)
