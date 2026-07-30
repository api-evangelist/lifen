---
name: Send a medical document to a healthcare professional over MSSanté
description: Look up a recipient practitioner or structure in the MSSanté directory, send a medical
  document to them through the Lifen Platform, then poll the tracking endpoint for delivery status.
api: openapi/lifen-fhir-api-openapi.json
operations:
  - authenticate-application
  - search-healthcare-professional
  - search-healthcare-structure
  - send-document-to-the-hcp
  - send-to-mss-check-sending-status
scope: SEND_TO_MSS
generated: '2026-07-19'
method: generated
source: https://developer.lifen.fr/docs/send-a-document-to-healthcare-professional
---

# Send a medical document over MSSanté

MSSanté is the French national secure health-messaging network. Lifen's MSS API Service lets an
e-health application post a document to a healthcare professional or structure on that network.

**This flow writes and transmits protected health information to a third party. It is not
reversible. Require explicit human confirmation of the recipient and the document before calling the
send operation.**

## Before you start

- You need the `SEND_TO_MSS` functional scope on the token.
- Test against `https://api.post-prod.lifen.fr/fhir/v3` with audience
  `https://post-prod.platform-apis/` before touching production.

## Step 1 — get a token (`authenticate-application`)

`POST https://authentication.lifen.fr/v1/token` with `client_id`, `client_secret`, `audience`,
`database_reference`, `grant_type=client_credentials`. Reuse the token for its 172800-second
lifetime; send it as `Authorization: Bearer {access_token}`.

## Step 2 — resolve the recipient

Never guess an MSSanté address. Resolve it from the directory.

- `search-healthcare-professional` — `POST /v3/Practitioner/$search-for-mss` returns matching
  practitioners with their MSSanté addresses.
- `search-healthcare-structure` — `POST /v3/Organization/$search-for-mss` does the same for a
  healthcare structure, when the recipient is an organisation rather than a named individual.

Both return a FHIR `Bundle`. If the search returns more than one plausible match, stop and ask a
human which one to use — sending a patient's record to the wrong practitioner is a data breach, not
a retryable error.

## Step 3 — resolve the sender

The sender must also be a real MSSanté identity. Use the same
`search-healthcare-professional` operation to look up the sending practitioner.

## Step 4 — send the document (`send-document-to-the-hcp`)

`POST /v3/CommunicationRequest/$send-to-mss`. The request carries the sender, the recipient(s), the
document metadata (`DocumentReference`) and the document payload (`Binary`).

A successful call returns `201` with an extension holding the tracking URL, shaped like:

```
https://api.lifen.fr/fhir/v3/CommunicationRequest/$tracking?transaction-id={transaction-id}
```

**Persist that `transaction-id` before doing anything else.** There is no idempotency key on this
API — if you lose the transaction-id and retry the send, you will send the document twice.

Failure modes here:

- `412` — a precondition failed, typically an invalid parameter value.
- `422` — the resource violated a FHIR profile or a Lifen business rule; read
  `issue[].diagnostics` on the `OperationOutcome`.

## Step 5 — track delivery (`send-to-mss-check-sending-status`)

`GET /v3/CommunicationRequest/$tracking?transaction-id={transaction-id}`.

Poll this rather than assuming a `201` means delivered — the `201` only means Lifen accepted the
request. Respect the 10 requests/second platform limit; poll with backoff, not in a tight loop.

## Staying inside the rules

- Global rate limit: 10 requests/second. `429` on exceed.
- No idempotency key exists — dedupe on your side, keyed on your own document identifier, before
  calling `$send-to-mss`.
- Every error body is a FHIR `OperationOutcome` (`severity`, `code`, `diagnostics`), not
  problem+json.
