---
name: Score an ecommerce order and report its outcome
description: >-
  Submit an ecommerce order to Fraud.net for a real-time risk score before payment
  authorization, act on the score, then relay the eventual outcome so the models keep
  learning.
api: openapi/fraud-net-public-apis-openapi.json
operations:
  - POST_v2-risk-order-ecommerce
  - PATCH_risk-order
generated: '2026-09-10'
method: generated
source: openapi/fraud-net-public-apis-openapi.json
---

# Score an ecommerce order and report its outcome

## Before you start

- **Auth.** Every request carries `Authorization: Basic <base64(key:secret)>`. The key comes
  from the Developer section of the Fraud.net Case Management Portal — there is no
  self-service signup, so if you do not already have a key you cannot complete this skill.
- **Host.** The only host in the published contract is the sandbox,
  `https://api-sandbox.c008-m008-us.fraud.net`. Production is per-tenant and per-region and
  is issued during onboarding; use the host your account was given.
- **Device data first.** Load the Device Fingerprint SDK on the checkout page before the
  order is placed (see `components/fraud-net-components.yml`). The provider's own rules say
  missing device data "significantly degrades signal strength" — the score you get without
  it is not the score the API is capable of.

## Step 1 — Check the order BEFORE authorization

`POST /v2/risk/order/ecommerce` — operationId **`POST_v2-risk-order-ecommerce`**

Call this before you authorize payment. Calling it afterwards still records the order but
throws away the decision the score was for.

Body (`schema-risk-order-request-ecommerce`) composes these objects:

| Field | Contents |
|---|---|
| `transaction` | **Required**: `order_id`, `ordered_on`. Plus `order_total`, `order_currency`, `type`, `status`, `user_id`, `order_source`, `order_count`, `total_spent` |
| `payment` | Card network `type`, `method`, `bin`, `avs_result_code`, `cvv_result_code`, `3DS_eci`/`3DS_vid`/`3DS_xid`, `eci`, `emv_aid` |
| `device` | The `device_data` value the SDK wrote into your form, plus `ip_address`, `user_agent`, `fingerprint_id` |
| `billing`, `shipping` | Address objects (`address1`, `city`, `region`, `postal_code`, `country`, `email`, `phone`, names) |
| `products` | Line items |
| `delivery`, `campaign`, `promotions` | Courier/method/weight, UTM attribution, promo codes |

Send everything you have. Payload completeness is the single biggest driver of score
quality on this API.

## Step 2 — Read the result

A 200 returns `{success, data}`:

- `data.risk_score` — integer 0–100, 0 low risk, 100 high risk
- `data.risk_group` — one of `very low`, `low`, `medium`, `high`, `very high`
- `data.tags[]` — the rules and models that fired, each with `name`, `source`, `action`,
  `weight`, `risk_score`, `risk_group`
- `data.link` — URL to the record in the Case Management Portal
- `data.timer` — milliseconds Fraud.net spent scoring (the platform targets under 100ms)
- `data.id` — your own `order_id`, echoed

Fraud.net does not decide for you. Map `risk_group` (or a `risk_score` threshold) to
approve / review / decline in your own policy, and use `tags[]` to explain the call.

## Step 3 — Relay the outcome

`PATCH /v2/risk/order` — operationId **`PATCH_risk-order`**

Every time the order's status changes in your system — approved, shipped, returned,
charged back, confirmed fraud — send an Update. Required: `order_id`, `status`,
`agent_updated_on` (ISO 8601, UTC). Useful: `is_fraud`, `fraud_type`, `payment_status`,
`is_locked`, `is_fraud_cancel`, `cancelled_reasons`, `note`, `agent_code`, `agent_dept`.

PATCH is partial:

- **change** a value → include the field with the new value
- **clear** a value → send it as an empty string (`fraud_type: ""`)
- **leave alone** → omit the field entirely

## Error and retry rules

| Status | Meaning | What to do |
|---|---|---|
| 401 | Invalid credentials or Authorization header | Fix the Basic construction; do not retry blind |
| 403 | Key valid, not entitled to this surface | Stop; this is a commercial entitlement, not a bug |
| 404 | On Check, wrong path. On Update, `order_id` was never Checked | Send the Check first |
| 406 | Invalid data | Read `code` and `source` in the `{success, code, source, message}` envelope; fix the payload |
| **409** | **Duplicate** — same `order_id` on a Check, same `(order_id, updated_on)` on an Update | **Treat as ALREADY RECORDED. Never retry-loop.** The write landed; you will not get the original score back — read it from `data.link` |
| 500 / 502 / 503 / 504 | Server-side | Retryable with backoff — but re-read the 409 rule first |

**There is no `Idempotency-Key` header.** Replay protection is duplicate rejection on the
natural key, so a network timeout followed by a retry gets a 409, not the original body.
Persist the `order_id` you sent before you send it.

**There is no undo.** No cancel, void or delete operation exists. A submitted Check cannot
be withdrawn; you can only amend fields on it with a later Update. Do not submit
speculative Checks.

## Related

- `conventions/fraud-net-conventions.yml` — idempotency, reversibility, response envelope
- `errors/fraud-net-problem-types.yml` — the full published error vocabulary
- `rules/fraud-net-rules.yml` — the provider's own operational rules
