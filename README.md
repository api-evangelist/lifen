# Lifen

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
