---
name: Score a bank or processor transaction and report its outcome
description: >-
  Submit a banking / payment-processor transaction to Fraud.net for pre-authorization risk
  scoring, then relay the settlement, chargeback or fraud disposition back.
api: openapi/fraud-net-public-apis-openapi.json
operations:
  - check-transaction-bank
  - update-bank-transaction
generated: '2026-09-10'
method: generated
source: openapi/fraud-net-public-apis-openapi.json
---

# Score a bank or processor transaction and report its outcome

This is the Banking / Fintech surface — banks, PSPs, acquirers, processors and BaaS
platforms scoring a payment rather than a retail order.

## Auth and host

`Authorization: Basic <base64(key:secret)>`, key from the Case Management Portal.
Published host is the sandbox, `https://api-sandbox.c008-m008-us.fraud.net`; use the
production host issued to your tenant.

## Step 1 — Check the transaction

`POST /v2/risk/transaction/bank` — operationId **`check-transaction-bank`**

Body (`schema-risk-transaction-request-bank`) carries six objects:

| Field | Contents |
|---|---|
| `transaction` | **Required** `order_id`, `ordered_on`; plus amount, currency, type, status, event |
| `account` | Account state — `avail_funds`, `credit_limit`, `is_active`, `status`, and the change timestamps `pin_change_on`, `email_change_on`, `address_change_on`, `password_change_on`, `phone_change_on` |
| `customer` | Identifying and contact fields for the account holder |
| `payment` | `type` (visa/mc/amex/discover/diners_club/other), `method`, `bin`, `avs_result_code`, `cvv_result_code`, `3DS_eci`/`3DS_vid`/`3DS_xid`, `emv_aid`, `emv_chip_id`, `terminal_type`, `terminal_method`, `iban`, `swift_code` |
| `seller` | Counterparty merchant — `name`, `company`, `mcc_id`, `industry`, address |
| `device` | Fingerprint SDK payload, `ip_address`, `ip_type`, `user_agent` |

Those account-change timestamps matter: a payment shortly after a password or phone
change is the classic account-takeover pattern and the models weight it.

## Step 2 — Read the result

Same envelope as every Check: `data.risk_score` (0–100), `data.risk_group`, `data.tags[]`
(the rules that fired, with `weight` and `action`), `data.link` into the portal,
`data.timer` in milliseconds.

## Step 3 — Relay the disposition

`PATCH /v2/risk/transaction/bank` — operationId **`update-bank-transaction`**

Required: `order_id`. Send `status`, `updated_on`, `agent_updated_on`, and the disposition
fields as they become known: `is_fraud`, `fraud_type`, `is_locked`, `order_total`,
`event`, `agent_code`, `agent_dept`, `batch_code`, `note`. Nested `account`, `payment` and
`seller` patches carry the payment-side outcome — `payment.chargeback_status`,
`payment.payment_status`, `payment.card_status`, `payment.auth_code`.

Chargebacks are the highest-value signal you can send back. Send them.

## Idempotency and retries

Uniqueness on this Update is the **pair** `(order_id, updated_on)`, so two genuinely
different updates to the same transaction must carry different `updated_on` values or the
second is rejected as a duplicate with **409**.

409 means already recorded — stop, do not retry. 401/403 are auth and entitlement.
404 on an Update means the Check never landed. 5xx are retryable with backoff.

## No reversal

There is no cancel, void or delete. A submitted Check is permanent and feeds model
training; only field-level amendment via Update is possible, and no window is published
for how late an amendment may arrive.

## Related

- `conventions/fraud-net-conventions.yml`
- `errors/fraud-net-problem-types.yml`
- `data-model/fraud-net-data-model.yml`
