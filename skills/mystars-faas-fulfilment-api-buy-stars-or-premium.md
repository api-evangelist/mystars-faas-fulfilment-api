---
name: Buy Telegram Stars or Premium for a username
description: Quote, pre-flight the recipient, create an order, and pay it on-chain - the marquee MyStars FaaS flow.
api: openapi/mystars-faas-fulfilment-api-openapi-original.json
operations: [getPricing, checkRecipient, createOrder, getOrder]
generated: '2026-09-03'
method: generated
---

# Buy Telegram Stars or Premium for a username

Authenticate every request with the `X-Api-Key` header (key issued in the @my_stars_tg_bot
Telegram bot). Base URL: `https://api.mystars.tg`.

1. **Quote** — `getPricing` (`GET /v1/pricing`) with the product `type` (`stars` or `premium`),
   quantity/months, and payment currency. The quote carries `quoted_at` / `valid_until` — re-quote
   when it goes stale. Amounts are decimal strings; never parse them into floats.
2. **Pre-flight the recipient** — `checkRecipient` (`POST /v1/recipients/check`) with the target
   `@username`. This is free and creates nothing. If `eligible` is false, stop: creating the order
   would fail with `422 recipient_ineligible` anyway (also free, but pointless). If `indeterminate`
   is true, eligibility could not be decided right now — retry or warn the user.
3. **Create the order** — `createOrder` (`POST /v1/orders`). REQUIRED: a fresh, unique
   `Idempotency-Key` header. The response `payment` block says exactly how much to send, to which
   address, and with which `memo` (the order id). Optionally pass a `callback_url` to receive the
   signed terminal webhook.
   - On `503 unavailable`: retry shortly with the SAME `Idempotency-Key` — no order was created.
   - NEVER retry with a new key after an ambiguous failure — a new key creates a brand-new order
     and a new charge. Reuse the same key to re-query the original.
4. **Pay** — send the exact `amount` to `pay_to_address` with the exact `memo`, in the quoted
   currency, before `expires_at` (2-hour payment window). A wrong or missing memo ends the order
   `failed` and the funds are auto-reversed minus the network fee. Payment within the -1%..+2%
   tolerance is treated as exact.
5. **Confirm** — poll `getOrder` (`GET /v1/orders/{id}`) or wait for the webhook until a terminal
   status: `delivered`, `failed`, `reversed`, or `expired`. `held` is NOT terminal — keep waiting;
   do not re-create the order.

Rate limits: reads share a 60 req/min budget; pricing and recipient checks carry their own tighter
60 req/min probe cap; order create/poll/cancel has its own separate bucket. Honor `Retry-After` on
any 429.
