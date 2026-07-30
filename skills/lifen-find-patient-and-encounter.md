---
name: Find a patient and their hospital stay in Lifen
description: Authenticate against the Lifen Platform, search a hospital's patient index by name or
  identifier, then retrieve that patient's encounters (séjours) and insurance coverage.
api: openapi/lifen-fhir-api-openapi.json
operations:
  - authenticate-application
  - patient
  - access-patient-by-id
  - encounter_search
  - access-encounter-by-id
  - access-coverage-patient-information
scope: PATIENT_GAM_SEARCH
generated: '2026-07-19'
method: generated
source: https://developer.lifen.fr/docs/access-patient-information
---

# Find a patient and their hospital stay in Lifen

Lifen exposes a French hospital's information system as HL7 FHIR R4. Everything below runs against
one hospital at a time: the access token is bound to a healthcare organisation by
`database_reference`, so you never pick a tenant on the request.

## Before you start

- Get `client_id`, `client_secret` and `database_reference` from the Lifen account manager. There is
  no self-service signup.
- Work in the test environment first: `https://api.post-prod.lifen.fr/fhir/v3` with audience
  `https://post-prod.platform-apis/`.
- You need the `PATIENT_GAM_SEARCH` functional scope. Without it, reads return 403 with an
  OperationOutcome whose `diagnostics` reads "Operation forbidden (insufficient scope)".
- This is protected health information. Do not log response bodies.

## Step 1 — get a token (`authenticate-application`)

`POST https://authentication.lifen.fr/v1/token` with `client_id`, `client_secret`, `audience`,
`database_reference` and `grant_type=client_credentials`.

The response carries `access_token`, the granted `scope` string, and `expires_in` (172800 seconds).
Cache the token for its lifetime; do not re-authenticate per request — the platform allows only 10
requests/second in total.

Send it on every subsequent call as `Authorization: Bearer {access_token}`.

## Step 2 — search the patient (`patient`)

`POST /v3/Patient/_search` with `Content-Type: application/x-www-form-urlencoded`.

Useful parameters: `name`, `family`, `given`, `birthdate`, `identifier`, `phone`, `_id`
(multivalued), `_count` (max 100, `0` is rejected), `_sort` (default `_lastUpdated`).

The response is a FHIR `Bundle` of type `searchset` with `total`, `link` and `entry[]`. **This
service caps a search at 10 results** — narrow with `birthdate` or `identifier` rather than paging
for more.

Two identifiers matter on the returned Patient: the hospital's **IPP**, always present, and the
national **INS/NIA**, which Lifen exposes only when the patient's INS is qualified. Do not treat a
missing INS as an error.

Indexing is asynchronous — a patient created seconds ago may not be searchable yet. Fall back to
`access-patient-by-id` when you already hold the id.

## Step 3 — read the patient by id (`access-patient-by-id`)

`GET /v3/Patient/{id}`. Returns 404 if the id is unknown, 415 if the requested representation is
unsupported.

## Step 4 — find the stay (`encounter_search`)

`POST /v3/Encounter/_search`, same form encoding. Search by the patient reference and, when you want
a window rather than a day, use the FHIR date prefixes `gt`, `lt`, `ge`, `le` on the date parameter
(for example `date=ge2023-10-01&date=le2023-10-12`). This service is also capped at 10 results.

`GET /v3/Encounter/{id}` (`access-encounter-by-id`) reads a single stay.

## Step 5 — coverage, if you need it (`access-coverage-patient-information`)

`POST /v3/Coverage/_search` returns the patient's insurance coverage under the same
`PATIENT_GAM_SEARCH` scope.

## Paging

When you do need more than one page, cursor-page rather than re-searching. Take the `next` link out
of the Bundle's `link[]` and `GET` it — it already carries `_id`, `_index` and `_searchAfter`. Use
`previous` (`_searchBefore`) to go back. Pagination is not available on the Ramsay and Vivalto
tenants.

## Errors

Every failure is a FHIR `OperationOutcome`, not RFC 9457 problem+json. Read `issue[].severity`,
`issue[].code` and `issue[].diagnostics`.

- `401` — token missing or expired; re-run step 1.
- `403` — the token lacks `PATIENT_GAM_SEARCH`.
- `412` — an invalid search parameter value, typically a malformed date.
- `422` — the request violated a FHIR profile or a server business rule; the reason is in
  `diagnostics`.
- `429` — you exceeded 10 requests/second. Back off; no `Retry-After` is documented on API
  responses.

## Retry safety

Lifen publishes no idempotency key. These are all read operations, so retrying is safe.
