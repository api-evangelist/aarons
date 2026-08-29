---
name: aarons-hpp-autopay-retry
description: >-
  Retry a failed Aaron's EZPay automatic payment and update the customer's retry notification state.
api: aarons:aarons-hpp
spec: openapi/aarons-hpp-openapi.json
operations:
  - AutoPayCustomerRetry_Post
  - UpdateCustomerRetryNotification_Post
generated: '2026-08-29'
method: generated
source: openapi/aarons-hpp-openapi.json
---

# Aaron's HPP — EZPay retry

Two operations, both keyed on a single opaque correlator.

1. **`AutoPayCustomerRetry_Post` — `POST /AutoPayCustomerRetry`**
   Body: `AutoPayCustomerRetry` → `PaymentGuid`. Retries the failed automatic payment identified by
   that GUID. Returns a `ResponseStatus`-bearing response.

2. **`UpdateCustomerRetryNotification_Post` — `POST /UpdateCustomerRetryNotification`**
   Body: `UpdateCustomerRetryNotification` → `PaymentGuid`. Updates the notification state for the
   same payment. Response schema: `UpdateCustomerRetryNotificationResponse`.

## The thing to be careful about

`AutoPayCustomerRetry_Post` **moves money and has no idempotency key**. The contract documents no
`Idempotency-Key` header, no dedupe window, and no reversal operation. Calling it twice with the
same `PaymentGuid` has no declared behaviour — it may double-charge.

Consequently:

- Never retry this call on a network timeout. Treat a timeout as *unknown*, not as *failed*, and
  reconcile from the gateway postback (`FiservPostback` / `RepayAuthPostback`) before deciding.
- Keep your own dedupe key on `PaymentGuid` and refuse a second submission within your own window.
- There is **no documented way to void or refund** a retry that fired in error. See the
  `reversibility` block in `conventions/aarons-conventions.yml`.

## Errors

`ResponseStatus` (`ErrorCode`, `Message`, `Errors[]`) — no documented status codes and no error-code
registry. See `errors/aarons-problem-types.yml`.
