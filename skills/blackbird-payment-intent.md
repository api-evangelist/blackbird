---
name: blackbird-payment-intent
description: Accept a FLY payment from a Blackbird member with the Flynet Payment Intents API — create, confirm, and reconcile an intent idempotently, modeled on Stripe Payment Intents.
api: openapi/blackbird-flynet-openapi-original.yml
operations: [createPaymentIntent, confirmPaymentIntent, getPaymentIntent, cancelPaymentIntent, refundPaymentIntent]
method: generated
source: openapi/blackbird-flynet-openapi-original.yml + conventions/blackbird-conventions.yml + errors/blackbird-problem-types.yml
---

# Take a FLY payment on Flynet

Move FLY from a Blackbird member's wallet to a merchant, on behalf of an
approved partner. All Payment Intent routes require an **OAuth access token** —
API keys are not accepted. You also need your `flynet_merchant_id` (your third
credential, issued at payments onboarding, scoped to your client_id and
environment).

## Steps

1. `createPaymentIntent` — `POST /payment_intents` with body
   `{ flynet_merchant_id, customer_user_id, amount: { value, currency: "FLY" },
   idempotency_key, description }`. `value` is a stringified integer in FLY wei.
   - **Idempotency is required.** The key is unique per
     `(flynet_merchant_id, idempotency_key)`. First call → `201` new intent;
     replay with the same key → `200` same intent. Use your own order ID as the
     key so network retries never double-charge.
2. `confirmPaymentIntent` — `POST /payment_intents/{id}/confirm`. Transfers FLY.
   If the member lacks enough FLY it returns **400 `payment0030`** — pre-check
   with `listMyWallets`/`getBalance`. v1 cannot card-fund or auto-load FLY.
3. `getPaymentIntent` — `GET /payment_intents/{id}`. **There are no payment
   webhooks in v1** — poll this for the terminal `status`
   (`pending → paid`, or `canceled`/`refunded`/`expired`).
4. `cancelPaymentIntent` — `POST /payment_intents/{id}/cancel` (pending only).
5. `refundPaymentIntent` — `POST /payment_intents/{id}/refund` (paid only; full
   refunds only in v1).

## Rules

- Status is computed at read time from timestamps — you never set it directly;
  call cancel/confirm/refund and the status follows.
- Confirm/cancel/refund idempotency is state-based (re-confirming a paid intent
  returns it unchanged); no separate key needed.
- `paymentIntent0003` means the intent is not in a state that allows the
  operation. On successful confirm the member gets a receipt email from
  `flynet@blackbird.xyz`.
