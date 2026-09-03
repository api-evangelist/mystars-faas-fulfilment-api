---
name: Receive and verify MyStars order-status webhooks
description: Verify X-Faas-Signature HMAC callbacks, handle retries and secret rotation correctly.
api: openapi/mystars-faas-fulfilment-api-openapi-original.json
operations: [createOrder, orderStatusWebhook]
generated: '2026-09-03'
method: generated
---

# Receive and verify MyStars order-status webhooks

Webhooks are per order: pass a `callback_url` in `createOrder` (`POST /v1/orders`) and the signed
terminal event (`orderStatusWebhook`) POSTs to exactly that URL when the order reaches
`delivered`, `failed`, `reversed`, or `expired`. There is no register-webhook endpoint; omit
`callback_url` and poll `getOrder` instead.

1. **Verify authenticity** — compute the hex HMAC-SHA256 of the EXACT raw request body under your
   webhook secret and compare it to `X-Faas-Signature` (the Stripe/GitHub signing scheme). The
   secret is shown once when you run `/api_start` in the Telegram bot — it is separate from the
   API key.
2. **Handle rotation** — after `/api_rotate_webhook` there is a 24-hour rollover during which the
   header carries MULTIPLE comma-separated signatures. Always split on commas and accept the
   request if ANY entry matches your secret.
3. **Respond fast** — reply `2xx` within 5 seconds (connect + headers + body each). Redirects are
   not followed; the callback_url must be the final destination.
4. **Expect retries** — a timeout or non-2xx is retried with exponential backoff, then
   dead-lettered. Body and signature are stable across retries, so idempotent processing keyed on
   `order_id` + status is safe.
