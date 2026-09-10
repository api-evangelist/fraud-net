---
name: Screen an account application and its logins
description: >-
  Score a new account application for identity and application fraud, relay the decision,
  and score subsequent logins for account-takeover risk.
api: openapi/fraud-net-public-apis-openapi.json
operations:
  - POST_v2-transaction-account-application
  - PATCH_v2-transaction-account-application
  - POST_v2-account-login
generated: '2026-09-10'
method: generated
source: openapi/fraud-net-public-apis-openapi.json
---

# Screen an account application and its logins

The Identity surface covers the two moments where an account is most at risk: when it is
opened, and when someone signs in to it.

## Auth and host

`Authorization: Basic <base64(key:secret)>`, key from the Case Management Portal.
Published host is the sandbox, `https://api-sandbox.c008-m008-us.fraud.net`.

## Step 1 — Screen the application at onboarding

`POST /v2/risk/transaction/account_application` — operationId
**`POST_v2-transaction-account-application`**

Body (`schema-application-request`) carries eight objects: `application`
(`application_id`, `created_on`, `created_by`, `status`, `event`, `source`, `type`),
`account`, `address`, `bank`, `identity` (`type`, `id`, `country`, `region`), `employer`,
`device` and `source` (attribution: `source`, `medium`, `campaign`, `term`, `content`,
`http_referrer`).

**Spec gap to plan around:** this operation declares only a `200` in the published
contract — no 4xx or 5xx. The shared error traits still apply in practice, so code
defensively against the same 401/403/404/406/409/5xx vocabulary the rest of the API
publishes, rather than assuming errors cannot occur.

## Step 2 — Relay the application decision

`PATCH /v2/risk/transaction/account_application` — operationId
**`PATCH_v2-transaction-account-application`**

Send `updated_on`, `status`, `queue_status`, `is_fraud`, `fraud_type`, `refuse_reason`,
`cancelled_reason`, `approved_amount`, `payment_status`. Approve/refuse outcomes are what
turn a screening call into training data.

## Step 3 — Score logins on the live account

`POST /v2/account/login` — operationId **`POST_v2-account-login`**

Body (`schema-login-request`) is deliberately small: `login`, `device`, `source`. The
device object is doing most of the work here, so the Device Fingerprint SDK must be on the
login page, not only on checkout.

This operation also declares only a `200` in the published contract.

## Rules that apply across all three

- **No `Idempotency-Key`.** The two Check operations here do not declare 409 either, so do
  not assume a duplicate is rejected — deduplicate on your own side using `application_id`
  before you call.
- **409 on the Update** means the same `(order_id, updated_on)` already landed. Already
  recorded; do not retry.
- **No undo.** Nothing removes a screening record.
- **PII.** These payloads carry regulated identity data. TLS only, and hold responses under
  your own retention policy — see `rules/fraud-net-rules.yml#pii-handling`.

## Related

- `conventions/fraud-net-conventions.yml`
- `errors/fraud-net-problem-types.yml`
- `conformance/fraud-net-conformance.yml`
