---
name: aarons-hpp-payment-session
description: >-
  Run Aaron's Hosted Payment Page flow end to end — open a session, attach customer context,
  capture device intelligence, tokenise and save the card, then authorise the payment.
api: aarons:aarons-hpp
spec: openapi/aarons-hpp-openapi.json
operations:
  - CreateSession2_Post
  - CustomerData2_Post
  - MemoryBearerToken_Post
  - MemoryTokenGuid_Post
  - SaveDeviceIntelligence_Post
  - CreateToken_Post
  - SaveToken_Post
  - AuthorizeSession_Post
generated: '2026-08-29'
method: generated
source: openapi/aarons-hpp-openapi.json
---

# Aaron's Hosted Payment Page — payment session

Base: `https://hpp.aarons.com` (from `host` + `basePath` in the published Swagger 2.0 document).

## Before you start

- **This is not a public API.** Every operation except `/ping` requires a bearer token in the
  `Authorization` header, and Aaron's issues no developer credentials. If you do not already hold a
  token from Aaron's, stop here — there is no self-service path.
- The contract declares `GET`, `PUT`, `POST` and `DELETE` on every path. That is a ServiceStack
  routing artefact, not a designed method surface. Use `POST` for the flow below; do not assume
  `DELETE` on a path means "undo".
- **There is no idempotency key.** Nothing in this contract makes a retried authorisation safe.
  Treat every write as at-most-once and reconcile out of band.

## Steps

1. **Open a session** — `CreateSession2_Post` `POST /CreateSession/2` with a `CreateSession` body
   carrying a `SessionItem`. `SessionItem` fields: `ApplicationId`, `BearerToken`, `ButtonText`,
   `DisplaySaveOnFile`, `EmailAddress`, `ExpiresAt`, `GCID`, `Lang`, `LangBuster`, `SaveOnFile`,
   `Store`, `TokenGuid`. The response (`CreateSessionResponse`) carries the `SessionGuid` every
   later call needs.

2. **Attach customer context** — `CustomerData2_Post` `POST /CustomerData/2` with `ApplicationId`,
   `Email`, `Language`, `SaveOnFile`, `SessionGuid`, `TermsVersion`, `TokenGuid`. `TermsVersion` is
   the customer's accepted terms revision — record which one you sent.

3. **Exchange short-lived credentials for the hosted page** — `MemoryBearerToken_Post`
   `POST /MemoryBearerToken` and `MemoryTokenGuid_Post` `POST /MemoryTokenGuid`. These return the
   `MemoryBearerTokenResponse` / `MemoryTokenGuidResponse` the browser-side page uses. No lifetime
   is documented; fetch them immediately before use.

4. **Capture device intelligence** — `SaveDeviceIntelligence_Post` `POST /SaveDeviceIntelligence`
   with `DeviceId`, `NavigatorData`, `TokenGuid`. This is the fraud-screening input; skipping it
   changes the risk posture of the authorisation, so do not skip it silently.

5. **Tokenise the card** — `CreateToken_Post` `POST /CreateToken`. The `Card` schema Aaron's uses
   downstream is `BIN`, `Brand`, `Last4`, `Name`, `Token`, `Masked`, `Exp` (`Expiration`),
   `Address1`, `City`, `Region`, `PostalCode`. **Never log or persist anything beyond
   `Token`, `Brand`, `Last4` and `Masked`.**

6. **Persist the instrument** — `SaveToken_Post` `POST /SaveToken` with `SessionGuid` and
   `TermsVersion`. Only do this when the customer chose `SaveOnFile` in step 1/2.

7. **Authorise** — `AuthorizeSession_Post` `POST /AuthorizeSession?SessionGuid=<guid>` with an
   `AuthorizeSession` body. Returns `AuthorizeSessionResponse`.

8. **Wait for the gateway callback.** The authorisation result arrives asynchronously as a postback
   from Fiserv or Repay — see `skills/aarons-hpp-gateway-postbacks.md`. Do not treat the
   `AuthorizeSession` 200 as settlement.

## Errors

The contract documents **only** `200` responses. In practice failures surface through the
ServiceStack `ResponseStatus` envelope reachable from the response schemas:
`ErrorCode`, `Message`, `StackTrace`, `Errors[]` (`ResponseError`: `ErrorCode`, `FieldName`,
`Message`, `Meta`), `Meta`. There is no published error-code registry — see
`errors/aarons-problem-types.yml`. Branch on `ErrorCode` being present, not on a status code alone.

## Reversibility

**None is documented.** No cancel, void, refund or reversal operation is described for any step
above, and no window is stated. Do not infer one from the `DELETE` verbs in the document. If a
payment must be reversed, that is an out-of-band process with Aaron's — see the `reversibility`
block in `conventions/aarons-conventions.yml`.
