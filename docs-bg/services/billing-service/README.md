# billing-service

> The token economy and payments: metering, balances, Stripe checkout, auto-top-up.

**Status:** 📋 planned · **Port (dev):** 8070 · **Database:** `billing`

## Responsibilities

- Token balances per tenant/user; **metering** by consuming `token.usage` events from
  model-gateway (with feature/agent attribution).
- Token pricing config; usage ledger + aggregation rollups; admin live usage view (SSE).
- Limits + alerts (low balance, daily budget) → `notification.requested` events.
- Token packages; **Stripe** Checkout; webhook handling (signature-verified, idempotent
  via stored event IDs); saved cards; **auto-top-up** at low balance.
- Welcome token bonus on `tenant.created`; limit-increase request flow (owner → admin).

## API sketch

`GET /balance` · `GET /usage` (+ admin SSE stream) · `GET /packages` · `POST /checkout` ·
`POST /webhooks/stripe` · `GET/POST /auto-topup` · `POST /limit-requests`

## Data owned

`balances`, `usage_ledger`, `token_pricing`, `packages`, `purchases`, `stripe_events`,
`auto_topup_settings`, `auto_topup_log`, `limit_requests`, `bonus_settings`.

## Dependencies

| Direction | What |
|---|---|
| Called by | agent-service (pre-flight balance check), platform-service, gateway (UI) |
| Calls | Stripe API |
| Events in | `token.usage` (metering), `tenant.created` (welcome bonus) |
| Events out | `notification.requested` (alerts), `audit.event` |
| Jobs | auto-top-up check, usage rollups |

## Design notes

- The monolith's payments module had **zero direct tests** (`CODE_QUALITY_REPORT.md`) —
  this rewrite treats Stripe flows as the highest-risk code in the platform: idempotent
  webhook processing, stored `stripe_events` dedupe, and a full test suite against
  stripe-mock are non-negotiable.
- Metering consumption must be idempotent (at-least-once delivery) — ledger entries keyed
  by event ID.
- **Pre-flight balance checks are advisory.** Metering arrives via events, so a balance
  read can lag the latest LLM calls by event-bus latency — a user near zero can briefly
  overspend by a few turns. Accepted: the overdraft is bounded by consumer lag and
  settles as soon as the `token.usage` consumer catches up.
- **Not carried over**: legacy plan-based billing, marketplace commission ledger
  ([04 §4](../../04-functional-coverage.md)).
- Migration note: Stripe data moves **last** (Phase 4) per
  [05 §5](../../05-migration-pros-and-cons.md).

## Implementation checklist

- [ ] Schema + ledger model (append-only, event-ID keyed)
- [ ] `token.usage` consumer with idempotency
- [ ] Balance reads + limits + alert thresholds
- [ ] Stripe checkout + webhook (signature verify, event dedupe) + saved cards
- [ ] Auto-top-up job + log
- [ ] Welcome bonus consumer; limit-request flow
- [ ] Admin usage SSE stream
- [ ] Full Stripe test suite (stripe-mock / test clocks)

## References

- [02 — service catalog entry](../../02-service-catalog.md#billing-service-8070)
- [06 §5.4 — why metering lives in model-gateway events](../../06-architectural-patterns.md)
- [07 §5.9 — dependency graph](../../07-dependency-graphs.md#59-billing-service)
