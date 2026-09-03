---
name: Manage and reverse MyStars orders
description: List, poll, cancel, and reason about terminal states and reversals safely.
api: openapi/mystars-faas-fulfilment-api-openapi-original.json
operations: [listOrders, getOrder, cancelOrder]
generated: '2026-09-03'
method: generated
---

# Manage and reverse MyStars orders

1. **List** — `listOrders` (`GET /v1/orders`) with cursor pagination: pass `limit`, then feed each
   response's `next_cursor` back as `cursor`.
2. **Poll one order** — `getOrder` (`GET /v1/orders/{id}`). Non-terminal: `awaiting_payment`,
   `held`. Terminal: `delivered`, `failed`, `reversed`, `expired`, `cancelled`.
3. **Cancel (the only agent-invoked reversal)** — `cancelOrder` (`POST /v1/orders/{id}/cancel`)
   works ONLY while the order is `awaiting_payment`, i.e. before payment lands and before
   `expires_at`. Any other state returns `409` — an order that is paid or processing cannot be
   cancelled. A duplicate cancel cannot double-fire: the second call gets the 409.
4. **After payment, reversal is automatic and provider-side** — you only ever pay for a successful
   delivery. Undeliverable orders end `reversed`; mismatched (underpaid/overpaid outside
   -1%..+2%) and unmatched (`no_memo`/`wrong_memo`) payments end `failed` — in every case funds
   return on-chain to the paying address minus the network fee, referenced in `reversal_tx`.
5. **If something looks stuck** — a `held` order is in progress or under manual review: give it
   time, keep polling. If it stays `held` for an extended period, or a terminal `reversed` order
   shows no on-chain reversal, contact support (https://t.me/Mystars_support_bot) with the
   `order_id`. Do NOT re-create the order with a new Idempotency-Key — that is a new charge.

Errors always arrive as `{ "error": { "code", "message" } }`; branch on `code`
(see errors/mystars-faas-fulfilment-api-problem-types.yml).
