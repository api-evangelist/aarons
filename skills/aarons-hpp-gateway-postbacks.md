---
name: aarons-hpp-gateway-postbacks
description: >-
  Handle the three inbound payment-gateway postbacks Aaron's Hosted Payment Page receives — Fiserv
  authorisation results, Repay authorisation events, and Repay card-vault lifecycle events.
api: aarons:aarons-hpp
spec: openapi/aarons-hpp-openapi.json
operations:
  - FiservPostback_Post
  - RepayAuthPostback_Post
  - RepayCardVaultPostback_Post
generated: '2026-08-29'
method: generated
source: openapi/aarons-hpp-openapi.json
---

# Aaron's HPP — gateway postbacks

**Direction matters.** These are callbacks Aaron's *receives* from its payment processors. They are
not events Aaron's emits to third-party subscribers, and there is no subscription API. This skill
describes the payload contracts so an agent working on Aaron's side of the integration can read
them correctly. See `asyncapi/aarons-hpp-webhooks.yml` for the structured catalog.

## `FiservPostback_Post` — `POST /FiservPostback`

Fiserv authorisation result. Payload (`FiservPostback`):

| Field | Meaning |
|---|---|
| `AuthCode` | Card-network authorisation code |
| `Card` | `BIN`, `Brand`, `Last4`, `Name`, `Token`, `Masked`, `Exp`, `Address1`, `City`, `Region`, `PostalCode` |
| `Error` | Gateway error text |
| `GatewayReason` | Gateway-side reason |
| `GatewayRefId` | Gateway reference — the correlator to your authorisation |
| `Reason` | Decline / result reason |
| `ZeroDollarAuth` | `CVV2` plus `AVS` (`StreetMatch`, `PostalCodeMatch`, `AssociationAvsResponse`) |

Correlate on `GatewayRefId`. Read `ZeroDollarAuth.AVS` before treating a card as verified —
`StreetMatch` and `PostalCodeMatch` are the address-verification outcome, and a zero-dollar
authorisation that succeeded with both mismatching is not the same as a verified card.

## `RepayAuthPostback_Post` — `POST /RepayAuthPostback`

Repay authorisation event, in Repay's envelope:

- `event_meta_data`: `version`, `event_type` — **this is the only versioned event envelope Aaron's
  publishes anywhere.** Branch on `event_type`; the value enumeration is not published, so treat
  unknown types as ignorable rather than as an error.
- `event_data` (`RepayAuthEventData`): `request`, `result`, `timestamp`.
  - `request`: `gateway_mid`, `amount`, `amount_cents`, `custom_fields`, `customer_id`,
    `invoice_number`, `address`, `card_brand`, `card_expiration`, `card_last_four`, `card_bin`,
    `card_name`, `card_type`, `nickname`, `payment_channel`.
  - `result`: `host_code`, `host_url`, `last_batch_number`, `message`, `message1`, `message2`,
    `original_transaction_id`, `pn_ref`, `resp_msg`, `result_code`, `auth_code`, `avs_result`,
    `commercial_card`, `cv_result`, `card_token`.

Prefer `amount_cents` over `amount` — integer minor units avoid the float rounding that `amount`
invites. Correlate on `original_transaction_id` / `pn_ref`.

## `RepayCardVaultPostback_Post` — `POST /RepayCardVaultPostback`

Stored-payment lifecycle. `event_data` (`CardVaultEventData`): `gateway_mid`, `custom_fields`,
`customer_id`, `customer_key`, `nickname`, `stored_payment_id`, `card_bin`, `card_brand`,
`card_expiration`, `card_last_four`, `card_type`, `customer_name`,
`is_eligible_for_disbursement`, `stored_payment_type`, `timestamp`.

`stored_payment_id` is the vault handle to reconcile against whatever `SaveToken_Post` stored.

## Security and hygiene

- Authentication on these paths is the same shared `Bearer` token as the rest of the contract. **No
  per-message signature scheme is documented** — no HMAC header, no timestamp tolerance, no replay
  window. Verify the transport and the correlator, and do not treat receipt alone as proof of origin.
- Every one of these payloads carries card metadata. Persist at most `card_brand`,
  `card_last_four`, `card_bin` and the token; never the full pan-adjacent set, never `custom_fields`
  blindly.
- No redelivery or retry policy is published. Make your handler idempotent on
  `GatewayRefId` / `original_transaction_id` / `stored_payment_id` yourself — the contract will not
  do it for you.
