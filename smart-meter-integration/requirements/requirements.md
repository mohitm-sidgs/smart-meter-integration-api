# Requirements Document
## Smart Meter Integration — Customer Energy Insights Portal

| Field             | Detail                                              |
|-------------------|-----------------------------------------------------|
| Project Name      | Smart Meter Integration                             |
| Interface/API     | Customer Energy Insights API Suite                  |
| Source System     | AMI Platform (Advanced Metering Infrastructure)     |
| Target System     | Customer Energy Insights Portal                     |
| Repository        | mohitm-sidgs/smart-meter-integration-api            |
| Branch            | develop                                             |
| Document Version  | 0.1 (Working Draft)                                 |
| Prepared By       | Requirements Intelligence Agent                     |
| Date              | 26-Jun-2026                                         |
| Status            | Pending Human Review                                |

---

## 1. Business Context

### 1.1 Business Objective
The Customer Energy Insights Portal is scheduled to go live in Q4. Currently,
there is no integration layer connecting the Advanced Metering Infrastructure
(AMI) platform to the portal. Customers are unable to view their own usage
data. This integration will close that gap by delivering three headline
features at go-live:

1. Near real-time electricity consumption (data no older than 1 hour)
2. Historical usage comparison (12 months monthly totals + 30 days interval data)
3. Projected billing cost for the current billing period

### 1.2 Stakeholders
| Name             | Role                          |
|------------------|-------------------------------|
| Sarah Chen       | Project Lead / Business Owner |
| James Holloway   | Business Requirements Owner   |
| Priya Nair       | Integration Architect         |
| David Okafor     | AMI Platform SME              |
| Tom Reid         | Security & Compliance         |
| Michelle Torres  | Salesforce / SAP SME          |

---

## 2. Scope

### 2.1 In Scope
| # | API Name                        | Layer      | Build Type          |
|---|---------------------------------|------------|---------------------|
| 1 | Smart Meter System API          | System     | New Build           |
| 2 | Customer Account Service API    | System     | Reuse (Exchange)    |
| 3 | SAP Rate Enquiry System API     | System     | New Build           |
| 4 | Customer Energy Process API     | Process    | New Build           |
| 5 | Customer Energy Experience API  | Experience | New Build           |

### 2.2 Out of Scope
- AMI OAuth migration (separate workstream — see Risk R-01)
- Customer-facing portal frontend development
- Smart meter hardware / device management
- Billing system write-back or invoice generation

---

## 3. Integration Architecture

### 3.1 API-Led Connectivity Model
This integration follows the PG&E mandated MuleSoft API-led connectivity
model with three distinct layers:

```
[Portal / Frontend]
        │  JWT Token (validated at E-API edge only)
        ▼
[Customer Energy Experience API]         ← Experience Layer
        │  OAuth Client Credentials (platform identity)
        ▼
[Customer Energy Process API]            ← Process Layer
        ├──► [Customer Account Service API]   ← System Layer (Salesforce) — REUSE
        ├──► [Smart Meter System API]         ← System Layer (AMI Platform) — NEW
        └──► [SAP Rate Enquiry System API]    ← System Layer (SAP) — NEW
```

### 3.2 Architectural Constraints
- The Smart Meter System API must never be called directly from the portal.
  All requests must flow through the Process API to ensure the Salesforce
  authorisation check is always enforced.
- No layer may bypass another layer per PG&E API-led connectivity mandate.
- The customer JWT token must be validated at the Experience API edge only
  and must never be propagated to downstream layers.
- Downstream calls from the Process API must use the platform's own OAuth
  client credentials — never the customer's token.
- Raw interval data must not be cached or persisted anywhere in the
  integration layer. The AMI platform is the system of record.

---

## 4. Business Requirements

| ID    | Requirement                                                                  |
|-------|------------------------------------------------------------------------------|
| BR-01 | Customers must be able to log into the portal and view their near            |
|       | real-time electricity consumption. Data must be no more than 1 hour old.    |
| BR-02 | Customers must be able to view a historical usage comparison showing         |
|       | 12 months of monthly totals and 30 days of 15-minute interval data.         |
| BR-03 | Customers must be able to view a projected billing cost for the current      |
|       | billing period based on consumption to date and their active tariff rate.   |
| BR-04 | Customers must only ever be able to see their own meter data. Cross-account  |
|       | data access is prohibited under California privacy law.                     |
| BR-05 | The integration must be live and operational by Q4 (Customer Energy          |
|       | Insights Portal go-live date).                                              |

---

## 5. Functional Requirements

### 5.1 Smart Meter System API (New Build)

| ID     | Requirement                                                                 |
|--------|-----------------------------------------------------------------------------|
| FR-101 | The System API must wrap the AMI platform's internal REST API and expose   |
|        | a stable, consumer-facing RAML-first interface.                            |
| FR-102 | The API must accept a meter ID and a date range as input parameters.       |
| FR-103 | The API must filter the AMI response to return consumption-relevant fields  |
|        | only. Device diagnostics, tamper flags, and operational metadata must be   |
|        | excluded from all responses.                                               |
| FR-104 | The API must handle AMI pagination internally for requests spanning more    |
|        | than 7 days of interval data. The AMI platform enforces a soft limit of    |
|        | 7 days per single request. This pagination must be transparent to callers. |
| FR-105 | The API must return interval readings at 15-minute granularity.            |
| FR-106 | The API must authenticate to the AMI platform using Basic Auth credentials  |
|        | stored in Key Vault. These credentials must never be propagated beyond     |
|        | the System API layer.                                                      |
| FR-107 | A RAML 1.0 contract must be authored and published to Anypoint Exchange    |
|        | before implementation begins (RAML-first mandate).                         |

**AMI Platform Technical Notes:**
- Approximately 4 million meters in the service territory transmit 15-minute
  interval data to the AMI platform.
- Data is typically available in the AMI API within 20–30 minutes of the
  interval ending. The freshest available data will be approximately
  30–45 minutes old — this meets the BR-01 requirement of data no older
  than 1 hour.

---

### 5.2 Customer Account Service API (Reuse from Exchange)

| ID     | Requirement                                                                 |
|--------|-----------------------------------------------------------------------------|
| FR-201 | The existing Customer Account Service API published in Anypoint Exchange   |
|        | must be consumed as-is. No modifications are permitted to the asset.       |
| FR-202 | The Process API must call this API to retrieve all meter IDs authorised    |
|        | for the requesting customer account before any AMI query is made.          |
| FR-203 | The authorised meter ID lookup chain is:                                   |
|        | Customer Account ID → Service Account → Service Points → Meter IDs.       |
| FR-204 | The API must also return the customer's rate plan code, which will be      |
|        | passed to the SAP Rate Enquiry System API for cost projection.             |

---

### 5.3 SAP Rate Enquiry System API (New Build)

| ID     | Requirement                                                                 |
|--------|-----------------------------------------------------------------------------|
| FR-301 | The System API must wrap the SAP billing system's BAPI for rate enquiry    |
|        | and expose it as a RESTful interface.                                      |
| FR-302 | The API must accept a rate plan code as input and return applicable tariff  |
|        | tier information including per-kWh pricing and time-of-use schedules.      |
| FR-303 | The API must authenticate to SAP using a dedicated technical service user   |
|        | account with read-only access to the rate enquiry BAPI only. Shared or     |
|        | admin accounts are strictly prohibited.                                    |
| FR-304 | SAP service account credentials must be stored in Key Vault.               |
| FR-305 | All calls to SAP must be audit logged, capturing: timestamp, rate plan     |
|        | code queried, correlation ID, and call outcome.                            |
| FR-306 | A RAML 1.0 contract must be authored and published to Anypoint Exchange    |
|        | before implementation begins.                                              |

---

### 5.4 Customer Energy Process API (New Build)

| ID     | Requirement                                                                 |
|--------|-----------------------------------------------------------------------------|
| FR-401 | The Process API must accept a customer account ID as the primary input.    |
| FR-402 | Before querying the AMI platform, the Process API must call the Customer   |
|        | Account Service API to retrieve the authorised meter IDs for the account.  |
|        | It must never accept a raw meter ID directly from the portal request.      |
| FR-403 | The Process API must support three logical operations aligned to the        |
|        | Experience API endpoints:                                                  |
|        | (a) Current Consumption — return the most recent available interval data.  |
|        | (b) Historical Trend — return 12 months of monthly totals AND 30 days of  |
|        |     15-minute interval data as separate response shapes.                   |
|        | (c) Billing Projection — return projected cost for the current billing     |
|        |     period based on consumption to date and tariff rate from SAP.          |
| FR-404 | For the 30-day interval view, the Process API must internally paginate AMI |
|        | calls (max 7 days per request) and aggregate results before returning a    |
|        | single clean response to the caller.                                       |
| FR-405 | The SAP Rate Enquiry call for billing projection must be treated as an     |
|        | optional enrichment. If the SAP System API is unavailable, the Process     |
|        | API must still return consumption data with the cost projection field      |
|        | omitted or flagged as unavailable. The API must not fail entirely.         |
| FR-406 | Correlation IDs must be generated at the Experience API and propagated     |
|        | through every downstream call — AMI, Salesforce, and SAP — to support     |
|        | end-to-end distributed tracing.                                            |
| FR-407 | The Process API must implement circuit breaker and retry patterns for all  |
|        | downstream System API calls per PG&E integration standards.               |
| FR-408 | A RAML 1.0 contract must be authored and published to Anypoint Exchange    |
|        | before implementation begins.                                              |

---

### 5.5 Customer Energy Experience API (New Build)

| ID     | Requirement                                                                 |
|--------|-----------------------------------------------------------------------------|
| FR-501 | The Experience API is the sole externally facing layer for this            |
|        | integration. All portal interactions must flow through this layer.         |
| FR-502 | The Experience API must validate the customer's JWT token at the edge.     |
|        | The customer JWT must never be propagated to downstream Process or System  |
|        | API layers under any circumstances.                                        |
| FR-503 | After JWT validation, all downstream calls must be made using the          |
|        | platform's OAuth client credentials.                                       |
| FR-504 | The Experience API must expose exactly three endpoints:                    |
|        | (a) GET /energy/current-consumption                                        |
|        |     Returns near real-time usage data.                                     |
|        | (b) GET /energy/historical-trend                                           |
|        |     Returns 12-month monthly totals and 30-day interval data.             |
|        | (c) GET /energy/billing-projection                                         |
|        |     Returns projected cost for the current billing period.                |
| FR-505 | Responses must be shaped precisely to the portal front end's requirements. |
|        | Raw Process API payloads must not be passed through unmodified.           |
| FR-506 | User-facing, channel-appropriate error messages must be returned to the    |
|        | portal. Internal stack traces or system error details must never be        |
|        | exposed to the consumer.                                                   |
| FR-507 | A RAML 1.0 contract must be authored and published to Anypoint Exchange    |
|        | before implementation begins.                                              |

---

## 6. Non-Functional Requirements

| ID     | Category        | Requirement                                                   |
|--------|-----------------|---------------------------------------------------------------|
| NFR-01 | Data Freshness  | Portal data must be no more than 1 hour old at time of       |
|        |                 | display. AMI data is available ~30–45 min after interval end.|
| NFR-02 | Data Retention  | Raw interval data must not be stored or cached in the        |
|        |                 | integration layer. AMI platform is the system of record.     |
| NFR-03 | Availability    | TBD — target availability SLA for portal-facing APIs.        |
| NFR-04 | Performance     | TBD — p95 response time targets for each API layer.          |
| NFR-05 | Scalability     | TBD — concurrent user and request throughput thresholds.     |
| NFR-06 | Rate Limiting   | TBD — throttling policy for portal traffic to AMI backend.   |
| NFR-07 | Security        | All APIs must enforce TLS 1.2 or above on all connections.   |
| NFR-08 | Observability   | Correlation IDs must be propagated through all layers to     |
|        |                 | support end-to-end distributed tracing and audit.            |
| NFR-09 | Audit Logging   | All SAP rate enquiry calls must be audit logged with         |
|        |                 | timestamp, rate plan code, correlation ID, and outcome.      |

---

## 7. Security Requirements

| ID     | Requirement                                                                  |
|--------|------------------------------------------------------------------------------|
| SEC-01 | Customer account authorisation must be enforced on every meter data query.  |
|        | A customer account ID must be validated against authorised meter IDs via    |
|        | Salesforce before the AMI platform is queried.                              |
| SEC-02 | Customers must only be able to access their own meter data. Cross-account   |
|        | data access must be technically prevented, not policy-controlled alone.     |
| SEC-03 | The customer JWT token must be validated at the Experience API layer only   |
|        | and must never propagate beyond it.                                         |
| SEC-04 | AMI platform Basic Auth credentials must be held exclusively within the     |
|        | Smart Meter System API and stored in Key Vault. Never exposed upstream.     |
| SEC-05 | The SAP technical service account must have minimum privilege — read-only   |
|        | access to the rate enquiry BAPI only. No shared or admin accounts.          |
| SEC-06 | All SAP service account credentials must be stored in Key Vault.            |
| SEC-07 | Correlation IDs must be propagated through every layer to support audit     |
|        | traceability for customer data access inquiries or complaints.              |

---

## 8. Integration Points & Affected Systems

| System           | Role                       | Protocol    | Auth Method          | Notes                           |
|------------------|----------------------------|-------------|----------------------|---------------------------------|
| AMI Platform     | Source of meter interval   | REST / JSON | Basic Auth           | Internal only. OAuth migration  |
|                  | data                       |             | (Key Vault)          | is a separate workstream.       |
| Salesforce       | Customer account &         | REST / JSON | OAuth 2.0            | Customer Account Service API    |
|                  | meter ID lookup            |             |                      | already in Exchange — reuse.    |
| SAP Billing      | Tariff rate lookup         | BAPI → REST | SAP Service User     | New System API wrapper needed.  |
|                  |                            |             | (Key Vault)          |                                 |
| Customer Portal  | Consumer of Energy APIs    | REST / JSON | JWT (PG&E online     | JWT validated at E-API edge     |
|                  |                            |             | account)             | only.                           |

---

## 9. Decisions & Rationale

| ID   | Decision                                               | Rationale                                           |
|------|--------------------------------------------------------|-----------------------------------------------------|
| D-01 | Customer Account Service API reused from Exchange      | Already published, validated, and production-ready. |
|      | as-is with no modifications.                          | Avoids duplication and aligns with reuse mandate.   |
| D-02 | SAP Rate Enquiry treated as optional enrichment in     | SAP is a new build and adds scope/delivery risk.    |
|      | the Process API MVP.                                  | Portal must degrade gracefully without cost data.   |
| D-03 | Process API handles AMI pagination internally for      | Simplifies portal integration — one request, one    |
|      | 30-day interval requests.                             | clean response. Portal complexity is abstracted.    |
| D-04 | Customer JWT validated at Experience API edge only.    | Customer token must never reach Salesforce or SAP.  |
|      | Downstream calls use platform OAuth credentials.      | Confirmed as the correct security pattern.          |
| D-05 | Raw interval data not cached in middleware.            | AMI is the system of record. Caching creates data   |
|      |                                                        | residency and California privacy law compliance risk.|

---

## 10. Assumptions

| ID   | Assumption                                                                   |
|------|------------------------------------------------------------------------------|
| A-01 | The Customer Energy Insights Portal authenticates customers via PG&E online  |
|      | account and issues a JWT token presented to the Experience API.             |
| A-02 | The AMI platform REST API is available and sufficiently performant to        |
|      | support portal-level traffic without a dedicated caching layer.             |
| A-03 | The existing Customer Account Service API in Exchange already supports the   |
|      | rate plan code field in its response. To be confirmed with the Salesforce   |
|      | team before implementation.                                                 |
| A-04 | The SAP BAPI for rate enquiry is accessible from the MuleSoft runtime        |
|      | network and does not require additional firewall changes.                   |
| A-05 | Key Vault is provisioned and accessible for storing AMI Basic Auth and SAP  |
|      | service account credentials.                                                |

---

## 11. Risks

| ID   | Risk                                               | Impact | Mitigation                                        |
|------|----------------------------------------------------|--------|---------------------------------------------------|
| R-01 | AMI platform uses Basic Auth — not the preferred   | High   | Credentials scoped to System API and stored in    |
|      | PG&E security standard. OAuth migration is a       |        | Key Vault. ARB exception or OAuth migration       |
|      | separate workstream with no confirmed timeline.    |        | timeline TBD.                                     |
| R-02 | SAP Rate Enquiry System API is a new build —       | High   | Treated as optional enrichment. Portal degrades   |
|      | adds scope and delivery risk for Q4 go-live.       |        | gracefully without cost projection if SAP is not  |
|      | Marketing has heavily promoted this feature.       |        | ready. Separate build track recommended.          |
| R-03 | AMI platform designed for internal operations      | Medium | System API acts as a traffic buffer. Rate limiting|
|      | tooling — may not withstand portal-scale traffic   |        | and throttling thresholds to be confirmed with    |
|      | across ~4 million meters.                          |        | AMI platform team. TBD.                           |
| R-04 | 12 months of 15-minute interval data would         | Medium | Business requirement scopes interval data to 30   |
|      | generate an unmanageable payload.                  |        | days only. Monthly totals used for 12-month trend.|

---

## 12. Open Questions

| ID    | Question                                                           | Owner           | Status |
|-------|--------------------------------------------------------------------|-----------------|--------|
| OQ-01 | What is the ARB exception status or OAuth migration timeline for   | David Okafor    | TBD    |
|       | the AMI platform Basic Auth?                                      |                 |        |
| OQ-02 | What are the p95 response time SLA targets for each API layer?    | Priya Nair      | TBD    |
| OQ-03 | What are the rate limiting / throttling thresholds for portal     | David Okafor    | TBD    |
|       | traffic to the AMI backend?                                       |                 |        |
| OQ-04 | What is the portal behaviour when AMI is unavailable?             | Sarah Chen      | TBD    |
|       | (fallback mode / degraded experience?)                            |                 |        |
| OQ-05 | Is the SAP Rate Enquiry System API confirmed in scope for Q4      | James Holloway  | TBD    |
|       | go-live or deferred to a later phase?                             |                 |        |
| OQ-06 | Does the existing Customer Account Service API in Exchange        | Michelle Torres | TBD    |
|       | already return the rate plan code field in its response?          |                 |        |

---

## 13. Action Items

| ID    | Action                                                  | Owner            | Due               |
|-------|---------------------------------------------------------|------------------|-------------------|
| AI-01 | Review and approve this requirements document.          | All Stakeholders | End of next week  |
| AI-02 | Confirm rate plan code availability in the Customer     | Michelle Torres  | Before design     |
|       | Account Service API response.                          |                  |                   |
| AI-03 | Confirm AMI OAuth migration timeline or initiate ARB    | David Okafor     | Before design     |
|       | exception request for Basic Auth usage.                |                  |                   |
| AI-04 | Confirm SAP Rate Enquiry go-live scope with business    | James Holloway   | Before design     |
|       | (Q4 or phased delivery).                               |                  |                   |
| AI-05 | Author RAML 1.0 contracts for all 4 new APIs and        | Priya Nair       | Before dev start  |
|       | publish to Anypoint Exchange.                          |                  |                   |
| AI-06 | Provision Key Vault entries for AMI Basic Auth and      | Platform / Ops   | Before dev start  |
|       | SAP service account credentials.                       |                  |                   |

---

## 14. MuleSoft Naming Standards (PG&E AIDLC)

Per PG&E Enterprise Standards, the following naming conventions apply:

| Artifact                            | Name                                      |
|-------------------------------------|-------------------------------------------|
| Experience API Project              | customer-energy-eapi                      |
| Process API Project                 | customer-energy-papi                      |
| Smart Meter System API Project      | smart-meter-sapi                          |
| SAP Rate Enquiry System API Project | sap-rate-sapi                             |
| RAML — Experience API               | customer-energy-experience-api-v1.raml    |
| RAML — Process API                  | customer-energy-process-api-v1.raml       |
| RAML — Smart Meter System API       | smart-meter-system-api-v1.raml            |
| RAML — SAP Rate System API          | sap-rate-system-api-v1.raml               |
| Exchange Asset — E-API              | Customer Energy Experience API            |
| Exchange Asset — P-API              | Customer Energy Process API               |
| Exchange Asset — AMI S-API          | Smart Meter System API                    |
| Exchange Asset — SAP S-API          | SAP Rate Enquiry System API               |
| Base URI — E-API                    | /eapi/customer-energy/v1                  |
| Base URI — P-API                    | /papi/customer-energy/v1                  |
| Base URI — Smart Meter S-API        | /sapi/smart-meter/v1                      |
| Base URI — SAP Rate S-API           | /sapi/sap-rate/v1                         |

---

> ⚠️ **Notice:** This is a working draft (v0.1) generated by the Requirements
> Intelligence Agent from a meeting transcript dated 26-Jun-2026. It has not
> yet been reviewed or approved by the project team. All TBD items must be
> resolved before downstream design and development begins. Human review is
> required before this document is used as a basis for technical design.
