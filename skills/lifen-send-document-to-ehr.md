---
name: Send a medical document into a hospital EHR with Lifen
description: Push a medical document into a hospital's Electronic Health Record (DPI) through the
  Lifen Platform, attached to the right patient and stay, then confirm integration via the tracking
  endpoint.
api: openapi/lifen-fhir-api-openapi.json
operations:
  - authenticate-application
  - patient
  - encounter_search
  - communicationrequestsend-to-ehr-1
  - send-to-mss-check-sending-status
scope: SEND_TO_EHR
generated: '2026-07-19'
method: generated
source: https://developer.lifen.fr/docs/send-a-document-to-the-ehr
---

# Send a medical document into a hospital EHR

The Send-documents-to-EHR API Service delivers a document into a hospital's Electronic Health Record
(DPI in French) as an HL7 ORU message. The document lands in a real patient's chart — treat this as
a write to a clinical system of record and require human confirmation before sending.

## Before you start

- You need the `SEND_TO_EHR` functional scope, which grants `ORGANIZATION_READ`, `BINARY_CREATE`,
  `COMMUNICATIONREQUEST_CREATE` and `DOCUMENTREFERENCE_CREATE`.
- Test environment: `https://api.post-prod.lifen.fr/fhir/v3` (Ramsay tenants use
  `https://api-ramsay.post-prod.lifen.fr/fhir/v3`).

## Step 1 — get a token (`authenticate-application`)

`POST https://authentication.lifen.fr/v1/token` with `client_id`, `client_secret`, `audience`,
`database_reference`, `grant_type=client_credentials`. Send it as `Authorization: Bearer {token}`.

## Step 2 — identify the patient and the stay

If your application already holds the hospital's IPP, use it. Otherwise resolve the patient with
`patient` (`POST /v3/Patient/_search`) and, when the document belongs to a specific stay, find it
with `encounter_search` (`POST /v3/Encounter/_search`). Both operations require the
`PATIENT_GAM_SEARCH` scope, which is granted separately from `SEND_TO_EHR` — if your application
only has `SEND_TO_EHR`, the patient reference must come from your own system.

Attaching a document to the wrong patient is the failure that matters here. Confirm the match with a
human when the search returned more than one candidate.

## Step 3 — send the document (`communicationrequestsend-to-ehr-1`)

`POST /v3/CommunicationRequest/$send-to-ehr`.

- Production: `https://api.lifen.fr/fhir/v3/CommunicationRequest/$send-to-ehr`
- Test: `https://api.post-prod.lifen.fr/fhir/v3/CommunicationRequest/$send-to-ehr`

The request bundles the `DocumentReference` metadata and the `Binary` payload. **The Binary must be
under 15 MB, and the receiving EHR may enforce a lower limit than that.** Check the size before
sending rather than handling a `413`.

A `201` response carries the tracking URL:

```
https://api.lifen.fr/fhir/v3/CommunicationRequest/$tracking?transaction-id={transaction-id}
```

Store the `transaction-id` immediately. There is no idempotency key on this API, so a blind retry
files the document twice in the patient's chart.

Failure modes:

- `401` — token missing or expired.
- `403` — the token lacks `SEND_TO_EHR`.
- `422` — the resource violated a FHIR profile or a Lifen business rule; read `diagnostics`.
- `413` — payload over 15 MB.

## Step 4 — confirm integration (`send-to-mss-check-sending-status`)

`GET /v3/CommunicationRequest/$tracking?transaction-id={transaction-id}` — the same tracking
operation serves both the EHR and MSSanté send paths. A `201` means Lifen accepted the request, not
that the DPI integrated the document. Poll tracking to confirm, with backoff inside the 10
requests/second platform limit.

## Reacting to the chart instead of polling

If you need to know when the hospital updates a patient or opens a stay, subscribe to webhooks
(`patient.updated`, `patient.merged`, `encounter.created`) rather than polling searches. See
`asyncapi/lifen-platform-webhooks.yml` — verify the `x-lifen-platform-signature` HMAC-SHA256 header
and allowlist Lifen's source IPs before trusting any delivery.
