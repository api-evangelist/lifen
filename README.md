# Lifen

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Lifen (Honestica SAS, Paris) is a French digital-health company. The Lifen Platform exposes HL7 FHIR
R4 APIs that let e-health applications exchange health data with French hospital information systems
and with healthcare professionals over the national MSSanté secure-messaging network.

- Website — https://www.lifen.fr/
- Developer portal — https://developer.lifen.fr/
- Status — https://status.lifen.fr/793930656
- GitHub — https://github.com/honestica

## API Services

| Service | What it does | Scope |
|---|---|---|
| Identity & Encounter | Patient identity, coverage and encounter (séjour) data inside a hospital | `PATIENT_GAM_SEARCH` |
| Appointment | Scheduled appointment data inside a hospital | `APPOINTMENT_GAM_SEARCH` |
| Send documents to EHR | Push a medical document into a hospital Electronic Health Record | `SEND_TO_EHR` |
| MSS | Send a medical document to healthcare professionals over MSSanté | `SEND_TO_MSS` |

Access is machine-to-machine over OAuth 2.0 client credentials; every token is bound to one
healthcare organisation via `database_reference`. A separate OIDC SSO API and a signed webhook
surface (patient/encounter events) complete the platform.

## Artifacts in this repo

| Dir | Artifact |
|---|---|
| `openapi/` | Harvested OpenAPI 3.1 for the FHIR API (10 operations) and the authentication API |
| `authentication/`, `scopes/` | OAuth2 client-credentials + OIDC profile, functional and technical scopes |
| `conventions/` | FHIR search, cursor pagination, error envelope, rate limiting, versioning |
| `errors/` | FHIR OperationOutcome envelope and the documented status-code catalog |
| `rate-limits/` | 10 req/s global, search result caps, 15 MB payload ceiling |
| `asyncapi/` | Webhook catalog — 3 event types, HMAC-SHA256 signing, retry rules, IP allowlist |
| `lifecycle/`, `changelog/` | Versioning (v3 current, v2 deprecated), status page, 8 dated entries |
| `conformance/` | FHIR R4, MSSanté, INS, OAuth2/OIDC/PKCE, RFC 8414, ISO 27001, HDS, GDPR |
| `sandbox/` | post-prod test environment, Postman collection, webhook test flow |
| `data-model/` | FHIR entity graph derived from the spec |
| `security/` | Probed TLS/HSTS/DNSSEC/SPF/DMARC posture |
| `skills/` | 3 Agent Skills grounded in real operationIds |
| `mcp/` | Candidate MCP tool surface derived from the OpenAPI |
| `well-known/`, `llms/` | RFC 8414 metadata, published llms.txt |

## Notes

- Lifen publishes **no first-party client SDKs, CLI or UI component library** — the honestica GitHub
  organisation is infrastructure and forks, not API clients.
- **No idempotency key** exists on the write operations; the platform's retry-safety story is the
  `$tracking` operation keyed on `transaction-id`.
- **No public vulnerability-disclosure policy, security.txt or bug-bounty program** was found.
  Certifications (ISO 27001, HDS) are published on the health-data security page.
- Credentials are issued by an account manager; there is no self-service sandbox signup.
