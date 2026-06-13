# agent-service

> The LangGraph agent runtime — the heart of the platform. Discovers agents from folders,
> runs their graphs, enforces the tool allow-list and write-approval interrupts, and owns
> conversation history (sessions + messages) in the same database.

**Status:** 📋 planned · **Port (dev):** 8020 · **Database:** `agent`

## Responsibilities

- Agent discovery: scan `app/agents/*/manifest.yaml` at startup, compile graphs, mount a
  single parametric route — **adding an agent never touches core code**.
- Default graph: load context → ReAct tool loop → approval interrupt for write tools.
- Shared tool catalog (`app/tools/`) with per-agent manifest allow-lists.
- Human-in-the-loop: write tools checkpoint to Postgres and pause; `/resume` continues.
- Context loading per turn: company profile, skills, AI memory, active project — driven by
  the manifest's `context:` block.
- AI memory (`remember` tool, directives) and skills (prompt snippets, folders, expiry).
- SSE streaming of tokens, tool events, and approval frames.
- **Conversations** (`conversations/` module): sessions per user/agent/channel, message
  history with paginated reads, attachment metadata, long-history summaries, TTL purge.

## API sketch

`GET /agents` (catalog, filterable by channel) · `POST /agents/{id}/chat` (SSE) ·
`POST /agents/{id}/resume` (approval decision) ·
`POST /agents/{id}/proactive` (ephemeral page summary — no session persisted) ·
`GET /sessions?user=&agent=` · `GET /sessions/{id}/messages?limit=` ·
`DELETE /sessions/{id}` · `GET/POST /memory` · `GET/POST /skills`

## Data owned

LangGraph checkpoints (interrupt/resume state), `sessions`, `messages`, `attachments`,
`ai_memory`, `user_skills`, `tool_validation_log`, `ai_traces`. No documents/vectors —
that's knowledge-service.

## Dependencies

| Direction | What |
|---|---|
| Calls | model-gateway (LLM), knowledge-service (retrieve), registry / business / document / integration / billing services (tools) |
| Called by | gateway (all chat traffic), future channel adapters |
| Events out | `audit.event` |
| Events in | `document.ingested` (context cache bust) |
| Jobs | temp-skill expiry, trace retention, session purge (retention, daily) |

## Design notes

- **This is the only fan-out hub** in the call matrix — keep every tool behind a port so
  targets can flip from monolith URLs to new services during migration.
- **Conversations are a module, not a service** — agent-service is their only consumer
  (channel adapters enter through the chat API, never the store), and history read/write
  happens on every chat turn, so a network hop here would be pure latency tax. The domain
  sees only the `ConversationStore` port; if a second consumer ever appears, extraction
  into a service is one adapter swap
  (see [02 § Deliberately merged](../../02-service-catalog.md#deliberately-merged-boundaries)).
- Keep the `conversations/` module boring: thin CRUD, no agent logic. Summarization,
  context windows, and prompt assembly stay in the agent domain. The `channel` field on
  sessions keeps a Telegram conversation and a web conversation with the same agent
  separate (or merged later, by product choice).
- Initial agents: a single `erp-agent` reproducing the monolith's global orchestrator;
  specialized agents (offer-agent, …) come after, as folders.
- Tool `kind="write"` ⇒ interrupt is enforced by the graph, never by prompt wording.
- Replaces the monolith's `workspace_sessions` / `workspace_messages` tables alongside the
  workspace chat machinery.

## Implementation checklist

- [ ] `Manifest` Pydantic model + discovery/registry with skip-on-broken-folder
- [ ] `AgentState` schema + `build_default_graph()` + `default_nodes()`
- [ ] Postgres checkpointer; `/resume` endpoint; approval SSE frames
- [ ] Idempotency keys for write tools (derived from checkpoint + approval ID — a crashed
      resume must never double-execute a write; see [03 § Pillar 5](../../03-agent-platform.md#pillar-5--write-actions-are-interrupts-not-trust))
- [ ] ToolBox: `specs(allowed)` / `dispatch()`; `ToolContext` with per-turn dedupe
- [ ] First tools: `knowledge_search`, `registry_query`, `registry_add_row` (write)
- [ ] Conversations module: schema (sessions, messages with role/tool_call_id,
      attachments), `ConversationStore` port + Postgres repository, atomic turn append,
      cursor pagination, retention job
- [ ] Ports + httpx adapters for model-gateway / knowledge / registry
- [ ] SSE protocol: token, tool_call, approval_required, done, error frames
- [ ] `erp-agent` manifest + system prompt (port monolith prompt v24 as baseline)
- [ ] Iteration cap + token budget guards
- [ ] Tenant scoping + RLS policies

## References

- [03 — Agent Platform & Extensibility](../../03-agent-platform.md) (the primary doc)
- [01 — key flows 1 & 2](../../01-architecture-overview.md#3-system-topology)
- [02 — service catalog entry](../../02-service-catalog.md#agent-service-8020)
- [06 §5 — agentic patterns](../../06-architectural-patterns.md)
- [07 §5.3 — dependency graph](../../07-dependency-graphs.md#53-agent-service)
