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
| Document Version  | 0.2 (Working Draft)                                 |
| Prepared By       | Requirements Intelligence Agent                     |
| Date              | 26-Jun-2026                                         |
| Status            | Pending Human Review                                |

---

## 1. Business Context

### 1.1 Business Objective
The Customer Energy Insights Portal is scheduled to go live in Q4. Currently, there is no integration layer connecting the Advanced Metering Infrastructure (AMI) platform to the portal. Customers are unable to view their own usage data. This integration will close that gap by delivering three headline features at go-live:

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
| 3 | Customer Energy Process API     | Process    | New Build           |
| 4 | Customer Energy Experience API  | Experience | New Build           |

### 2.2 Out of Scope
- AMI OAuth migration (separate workstream — see Risk R-01)
- Customer-facing portal frontend development
- Smart meter hardware / device management
- Billing system write-back or invoice generation
- **SAP Rate Enquiry System API — deferred post-Q4 go-live (confirmed out of scope by James Holloway)**

---

## 3. Integration Architecture

### 3.1 API-Led Connectivity Model
This integration follows the PG&E mandated MuleSoft API-led connectivity model with three distinct layers:

```
[Portal / Frontend]
        │  JWT Token (validated at E-API edge only)
        ▼
[Customer Energy Experience API]         ← Experience Layer
        │  OAuth Client Credentials (platform identity)
        ▼
[Customer Energy Process API]            ← Process Layer
        ├──► [Customer Account Service API]   ← System Layer (Salesforce) — REUSE
        └──► [Smart Meter System API]         ← System Layer (AMI Platform) — NEW
```

> ⚠️ **Note:** SAP Rate Enquiry System API has been confirmed out of scope for Q4 go-live and is excluded from the architecture diagram above. It will be added in a post-Q4 phase.

### 3.2 Architectural Constraints
- The Smart Meter System API must never be called directly from the portal. All requests must flow through the Process API to ensure the Salesforce authorisation check is always enforced.
- No layer may bypass another layer per PG&E API-led connectivity mandate.
- The customer JWT token must be validated at the Experience API edge only and must never be propagated to downstream layers.
- Downstream calls from the Process API must use the platform's own OAuth client credentials — never the customer's token.
- Raw interval data must not be cached or persisted anywhere in the integration layer. The AMI platform is the system of record.

---

## 4. Business Requirements

| ID    | Requirement                                                                  |
|-------|------------------------------------------------------------------------------|
| BR-01 | Customers must be able to log into the portal and view their near real-time electricity consumption. Data must be no more than 1 hour old. |
| BR-02 | Customers must be able to view a historical usage comparison showing 12 months of monthly totals and 30 days of 15-minute interval data. |
| BR-03 | Customers must be able to view a projected billing cost for the current billing period based on consumption to date and their active tariff rate. |
| BR-04 | Customers must only ever be able to see their own meter data. Cross-account data access is prohibited under California privacy law. |
| BR-05 | The integration must be live and operational by Q4 (Customer Energy Insights Portal go-live date). |

---

## 5. Functional Requirements

### 5.1 Smart Meter System API (New Build)

| ID     | Requirement                                                                 |
|--------|-----------------------------------------------------------------------------|
| FR-101 | The System API must wrap the AMI platform's internal REST API and expose a stable, consumer-facing RAML-first interface. |
| FR-102 | The API must accept a meter ID and a date range as input parameters. |
| FR-103 | The API must filter the AMI response to return consumption-relevant fields only. Device diagnostics, tamper flags, and operational metadata must be excluded from all responses. |
| FR-104 | The API must handle AMI pagination internally for requests spanning more than 7 days of interval data. The AMI platform enforces a soft limit of 7 days per single request. This pagination must be transparent to callers. |
| FR-105 | The API must return interval readings at 15-minute granularity. |
| FR-106 | The API must authenticate to the AMI platform using Basic Auth credentials stored in Key Vault. These credentials must never be propagated beyond the System API layer. |
| FR-107 | A RAML 1.0 contract must be authored and published to Anypoint Exchange before implementation begins (RAML-first mandate). |

**AMI Platform Technical Notes:**
- Approximately 4 million meters in the service territory transmit 15-minute interval data to the AMI platform.
- Data is typically available in the AMI API within 20–30 minutes of the interval ending. The freshest available data will be approximately 30–45 minutes old — this meets the BR-01 requirement of data no older than 1 hour.

---

### 5.2 Customer Account Service API (Reuse from Exchange)

| ID     | Requirement                                                                 |
|--------|-----------------------------------------------------------------------------|
| FR-201 | The existing Customer Account Service API published in Anypoint Exchange must be consumed as-is. No modifications are permitted to the asset. |
| FR-202 | The Process API must call this API to retrieve all meter IDs authorised for the requesting customer account before any AMI query is made. |
| FR-203 | The authorised meter ID lookup chain is: Customer Account ID → Service Account → Service Points → Meter IDs. |
| FR-204 | The API must also return the customer's rate plan code, which will be passed to the SAP Rate Enquiry System API for cost projection. |

---

### 5.3 SAP Rate Enquiry System API (New Build)

> ⚠️ **OUT OF SCOPE FOR Q4 GO-LIVE — This API has been confirmed as deferred to a post-Q4 phase. Requirements below are retained for future reference only.**

| ID     | Requirement                                                                 |
|--------|-----------------------------------------------------------------------------|
| FR-301 | The System API must wrap the SAP billing system's BAPI for rate enquiry and expose it as a RESTful interface. |
| FR-302 | The API must accept a rate plan code as input and return applicable tariff tier information including per-kWh pricing and time-of-use schedules. |
| FR-303 | The API must authenticate to SAP using a dedicated technical service user account with read-only access to the rate enquiry BAPI only. Shared or admin accounts are strictly prohibited. |
| FR-304 | SAP service account credentials must be stored in Key Vault. |
| FR-305 | All calls to SAP must be audit logged, capturing: timestamp, rate plan code queried, correlation ID, and call outcome. |
| FR-306 | A RAML 1.0 contract must be authored and published to Anypoint Exchange before implementation begins. |

---

### 5.4 Customer Energy Process API (New Build)

| ID     | Requirement                                                                 |
|--------|-----------------------------------------------------------------------------|
| FR-401 | The Process API must accept a customer account ID as the primary input. |
| FR-402 | Before querying the AMI platform, the Process API must call the Customer Account Service API to retrieve the authorised meter IDs for the account. It must never accept a raw meter ID directly from the portal request. |
| FR-403 | The Process API must support three logical operations aligned to the Experience API endpoints: (a) Current Consumption — return the most recent available interval data. (b) Historical Trend — return 12 months of monthly totals AND 30 days of 15-minute interval data as separate response shapes. (c) Billing Projection — return projected cost for the current billing period based on consumption to date and tariff rate from SAP. |
| FR-404 | For the 30-day interval view, the Process API must internally paginate AMI calls (max 7 days per request) and aggregate results before returning a single clean response to the caller. |
| FR-405 | The SAP Rate Enquiry call for billing projection must be treated as an optional enrichment. If the SAP System API is unavailable, the Process API must still return consumption data with the cost projection field omitted or flagged as unavailable. The API must not fail entirely. |
| FR-406 | Correlation IDs must be generated at the Experience API and propagated through every downstream call — AMI, Salesforce, and SAP — to support end-to-end distributed tracing. |
| FR-407 | The Process API must implement circuit breaker and retry patterns for all downstream System API calls per PG&E integration standards. |
| FR-408 | A RAML 1.0 contract must be authored and published to Anypoint Exchange before implementation begins. |

---

### 5.5 Customer Energy Experience API (New Build)

| ID     | Requirement                                                                 |
|--------|-----------------------------------------------------------------------------|
| FR-501 | The Experience API is the sole externally facing layer for this integration. All portal interactions must flow through this layer. |
| FR-502 | The Experience API must validate the customer's JWT token at the edge. The customer JWT must never be propagated to downstream Process or System API layers under any circumstances. |
| FR-503 | After JWT validation, all downstream calls must be made using the platform's OAuth client credentials. |
| FR-504 | The Experience API must expose exactly three endpoints: (a) GET /energy/current-consumption (b) GET /energy/historical-trend (c) GET /energy/billing-projection |
| FR-505 | Responses must be shaped precisely to the portal front end's requirements. Raw Process API payloads must not be passed through unmodified. |
| FR-506 | User-facing, channel-appropriate error messages must be returned to the portal. Internal stack traces or system error details must never be exposed to the consumer. |
| FR-507 | A RAML 1.0 contract must be authored and published to Anypoint Exchange before implementation begins. |

---

## 6. Non-Functional Requirements

| ID     | Category        | Requirement                                                   |
|--------|-----------------|---------------------------------------------------------------|
| NFR-01 | Data Freshness  | Portal data must be no more than 1 hour old at time of display. AMI data is available ~30–45 min after interval end. |
| NFR-02 | Data Retention  | Raw interval data must not be stored or cached in the integration layer. AMI platform is the system of record. |
| NFR-03 | Availability    | TBD — target availability SLA for portal-facing APIs. |
| NFR-04 | Performance     | TBD — p95 response time targets for each API layer. |
| NFR-05 | Scalability     | TBD — concurrent user and request throughput thresholds. |
| NFR-06 | Rate Limiting   | TBD — throttling policy for portal traffic to AMI backend. |
| NFR-07 | Security        | All APIs must enforce TLS 1.2 or above on all connections. |
| NFR-08 | Observability   | Correlation IDs must be propagated through all layers to support end-to-end distributed tracing and audit. |
| NFR-09 | Audit Logging   | All SAP rate enquiry calls must be audit logged with timestamp, rate plan code, correlation ID, and outcome. |

---

## 7. Security Requirements

| ID     | Requirement                                                                  |
|--------|------------------------------------------------------------------------------|
| SEC-01 | Customer account authorisation must be enforced on every meter data query. A customer account ID must be validated against authorised meter IDs via Salesforce before the AMI platform is queried. |
| SEC-02 | Customers must only be able to access their own meter data. Cross-account data access must be technically prevented, not policy-controlled alone. |
| SEC-03 | The customer JWT token must be validated at the Experience API layer only and must never propagate beyond it. |
| SEC-04 | AMI platform Basic Auth credentials must be held exclusively within the Smart Meter System API and stored in Key Vault. Never exposed upstream. |
| SEC-05 | The SAP technical service account must have minimum privilege — read-only access to the rate enquiry BAPI only. No shared or admin accounts. |
| SEC-06 | All SAP service account credentials must be stored in Key Vault. |
| SEC-07 | Correlation IDs must be propagated through every layer to support audit traceability for customer data access inquiries or complaints. |

---

## 8. Integration Points & Affected Systems

| System           | Role                       | Protocol    | Auth Method          | Notes                           |
|------------------|----------------------------|-------------|----------------------|---------------------------------|
| AMI Platform     | Source of meter interval data | REST / JSON | Basic Auth (Key Vault) | Internal only. OAuth migration is a separate workstream. |
| Salesforce       | Customer account & meter ID lookup | REST / JSON | OAuth 2.0 | Customer Account Service API already in Exchange — reuse. |
| SAP Billing      | Tariff rate lookup         | BAPI → REST | SAP Service User (Key Vault) | ⚠️ Out of scope for Q4. Deferred to post-Q4 phase. |
| Customer Portal  | Consumer of Energy APIs    | REST / JSON | JWT (PG&E online account) | JWT validated at E-API edge only. |

---

## 9. Decisions & Rationale

| ID   | Decision                                               | Rationale                                           |
|------|--------------------------------------------------------|-----------------------------------------------------|
| D-01 | Customer Account Service API reused from Exchange as-is with no modifications. | Already published, validated, and production-ready. Avoids duplication and aligns with reuse mandate. |
| D-02 | SAP Rate Enquiry treated as optional enrichment in the Process API MVP. | SAP is a new build and adds scope/delivery risk. Portal must degrade gracefully without cost data. |
| D-03 | Process API handles AMI pagination internally for 30-day interval requests. | Simplifies portal integration — one request, one clean response. Portal complexity is abstracted. |
| D-04 | Customer JWT validated at Experience API edge only. Downstream calls use platform OAuth credentials. | Customer token must never reach Salesforce or SAP. Confirmed as the correct security pattern. |
| D-05 | Raw interval data not cached in middleware. | AMI is the system of record. Caching creates data residency and California privacy law compliance risk. |
| D-06 | SAP Rate Enquiry System API confirmed out of scope for Q4 go-live. | Confirmed by James Holloway on 26-Jun-2026. Deferred to post-Q4 phase to protect Q4 delivery timeline. |

---

## 10. Assumptions

| ID   | Assumption                                                                   |
|------|------------------------------------------------------------------------------|
| A-01 | The Customer Energy Insights Portal authenticates customers via PG&E online account and issues a JWT token presented to the Experience API. |
| A-02 | The AMI platform REST API is available and sufficiently performant to support portal-level traffic without a dedicated caching layer. |
| A-03 | The existing Customer Account Service API in Exchange already supports the rate plan code field in its response. To be confirmed with the Salesforce team before implementation. |
| A-04 | The SAP BAPI for rate enquiry is accessible from the MuleSoft runtime network and does not require additional firewall changes. |
| A-05 | Key Vault is provisioned and accessible for storing AMI Basic Auth and SAP service account credentials. |

---

## 11. Risks

| ID   | Risk                                               | Impact | Mitigation                                        |
|------|----------------------------------------------------|--------|---------------------------------------------------|
| R-01 | AMI platform uses Basic Auth — not the preferred PG&E security standard. OAuth migration is a separate workstream with no confirmed timeline. | High | Credentials scoped to System API and stored in Key Vault. ARB exception or OAuth migration timeline TBD. |
| R-02 | ~~SAP Rate Enquiry System API is a new build — adds scope and delivery risk for Q4 go-live.~~ **[CLOSED FOR Q4]** | **Closed** | Confirmed out of scope for Q4. SAP Rate Enquiry System API deferred to post-Q4 phase. R-02 closed for Q4 delivery. |
| R-03 | AMI platform designed for internal operations tooling — may not withstand portal-scale traffic across ~4 million meters. | Medium | System API acts as a traffic buffer. Rate limiting and throttling thresholds to be confirmed with AMI platform team. TBD. |
| R-04 | 12 months of 15-minute interval data would generate an unmanageable payload. | Medium | Business requirement scopes interval data to 30 days only. Monthly totals used for 12-month trend. |

---

## 12. Open Questions

| ID    | Question                                                           | Owner           | Status | Resolution |
|-------|--------------------------------------------------------------------|-----------------|--------|------------|
| OQ-01 | What is the ARB exception status or OAuth migration timeline for the AMI platform Basic Auth? | David Okafor | TBD | — |
| OQ-02 | What are the p95 response time SLA targets for each API layer? | Priya Nair | TBD | — |
| OQ-03 | What are the rate limiting / throttling thresholds for portal traffic to the AMI backend? | David Okafor | TBD | — |
| OQ-04 | What is the portal behaviour when AMI is unavailable? (fallback mode / degraded experience?) | Sarah Chen | TBD | — |
| OQ-05 | Is the SAP Rate Enquiry System API confirmed in scope for Q4 go-live or deferred to a later phase? | James Holloway | **Resolved** | Confirmed out of scope for Q4 go-live. Deferred to post-Q4 phase. |
| OQ-06 | Does the existing Customer Account Service API in Exchange already return the rate plan code field in its response? | Michelle Torres | TBD | — |

---

## 13. Action Items

| ID    | Action                                                  | Owner            | Due               | Status |
|-------|---------------------------------------------------------|------------------|-------------------|--------|
| AI-01 | Review and approve this requirements document.          | All Stakeholders | End of next week  | Open |
| AI-02 | Confirm rate plan code availability in the Customer Account Service API response. | Michelle Torres | Before design | Open |
| AI-03 | Confirm AMI OAuth migration timeline or initiate ARB exception request for Basic Auth usage. | David Okafor | Before design | Open |
| AI-04 | Confirm SAP Rate Enquiry phased delivery timeline and requirements for post-Q4 implementation. | James Holloway | Post-Q4 Planning | Updated — SAP Rate Enquiry confirmed out of scope for Q4. Action updated to track post-Q4 delivery planning. |
| AI-05 | Author RAML 1.0 contracts for all 4 new APIs and publish to Anypoint Exchange. | Priya Nair | Before dev start | Open |
| AI-06 | Provision Key Vault entries for AMI Basic Auth and SAP service account credentials. | Platform / Ops | Before dev start | Open |

---

## 14. MuleSoft Naming Standards (PG&E AIDLC)

Per PG&E Enterprise Standards, the following naming conventions apply:

| Artifact                            | Name                                      |
|-------------------------------------|-------------------------------------------|
| Experience API Project              | customer-energy-eapi                      |
| Process API Project                 | customer-energy-papi                      |
| Smart Meter System API Project      | smart-meter-sapi                          |
| SAP Rate Enquiry System API Project | sap-rate-sapi *(post-Q4 — deferred)*      |
| RAML — Experience API               | customer-energy-experience-api-v1.raml    |
| RAML — Process API                  | customer-energy-process-api-v1.raml       |
| RAML — Smart Meter System API       | smart-meter-system-api-v1.raml            |
| RAML — SAP Rate System API          | sap-rate-system-api-v1.raml *(post-Q4 — deferred)* |
| Exchange Asset — E-API              | Customer Energy Experience API            |
| Exchange Asset — P-API              | Customer Energy Process API               |
| Exchange Asset — AMI S-API          | Smart Meter System API                    |
| Exchange Asset — SAP S-API          | SAP Rate Enquiry System API *(post-Q4 — deferred)* |
| Base URI — E-API                    | /eapi/customer-energy/v1                  |
| Base URI — P-API                    | /papi/customer-energy/v1                  |
| Base URI — Smart Meter S-API        | /sapi/smart-meter/v1                      |
| Base URI — SAP Rate S-API           | /sapi/sap-rate/v1 *(post-Q4 — deferred)*  |

---

## 15. Revision History

| Version | Date        | Author          | Changes                                                              |
|---------|-------------|-----------------|----------------------------------------------------------------------|
| 0.1     | 26-Jun-2026 | RIA Agent       | Initial draft generated from meeting transcript.                     |
| 0.2     | 26-Jun-2026 | RIA Agent       | SAP Rate Enquiry System API confirmed out of scope for Q4. Updated: Section 2.1 (removed from in scope), Section 2.2 (added to out of scope), Section 5.3 (deferred notice added), Section 11 R-02 (closed for Q4), Section 12 OQ-05 (resolved), Section 13 AI-04 (updated), Architecture diagram updated. New Section 15 (Revision History) added. |

---

> ⚠️ **Notice:** This is a working draft (v0.2) generated by the Requirements Intelligence Agent. It has not yet been reviewed or approved by the project team. All TBD items must be resolved before downstream design and development begins. Human review is required before this document is used as a basis for technical design.
