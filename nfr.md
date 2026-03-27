**Non-Functional Requirements (NFR) Document** for SmartData - the Teams-Based Data Explorer. This document defines the system's operational attributes, focusing on enterprise-grade security, performance, and reliability.

---

# Non-Functional Requirements (NFR) Specification

## 1. Security & Compliance (Priority: Critical)
Given that the solution acts as a conduit to sensitive Line-of-Business (LOB) data, security is the architectural foundation.

### 1.1. Authentication & Identity
* **NFR-SEC-01 (Single Sign-On):** The system MUST authenticate users exclusively via Microsoft Entra ID (Azure AD) using seamless SSO within Microsoft Teams. No secondary credential prompts shall be presented.
* **NFR-SEC-02 (Token Validation):** The API Gateway MUST validate the Entra ID JWT bearer token for valid signature, audience, expiration, and issuer on *every* request.

### 1.2. Authorization & Access Control
* **NFR-SEC-03 (Metadata RBAC):** Access to App Configurations, Entities, and Actions MUST be gated by Entra ID Group memberships evaluated at runtime.
* **NFR-SEC-04 (Database RLS Passthrough):** The API MUST inject the authenticated user’s Entra Object ID (`oid`) into the SQL Server context via `sp_set_session_context` prior to executing any query, ensuring native SQL Row-Level Security (RLS) is enforced.

### 1.3. Data Protection & Privacy
* **NFR-SEC-05 (Zero Data Persistence):** The SaaS platform MUST NOT store, cache, or log any LOB data retrieved from the customer’s SQL database.
* **NFR-SEC-06 (Encryption in Transit):** All data transmitted between the Teams client, API Gateway, and Customer SQL DB MUST be encrypted using TLS 1.2 or higher.
* **NFR-SEC-07 (Secret Management):** Customer database connection strings MUST be stored in Azure Key Vault and injected into the application memory at runtime. They must never be persisted in the metadata database in plaintext.

## 2. Performance & Scalability
The system must feel highly responsive, similar to a native application, despite fetching LOB data in real-time.

### 2.1. Response Times & Latency
* **NFR-PERF-01 (UI Rendering):** The Teams SPA MUST render the initial dashboard and UI components within **1 second** of the tab loading.
* **NFR-PERF-02 (API Gateway Overhead):** The API Gateway MUST process, authorize, and route incoming requests to the LOB database with a maximum internal processing overhead of **200 milliseconds** (excluding actual SQL execution time).
* **NFR-PERF-03 (Metadata Caching):** Configuration metadata (Views, Actions, UI mappings) MUST be cached in a distributed cache (e.g., Redis) to prevent repeated trips to the metadata database.

### 2.2. Scalability & Data Handling
* **NFR-SCAL-01 (Stateless Architecture):** The API Gateway MUST be entirely stateless to support horizontal scaling via Azure App Service / AKS auto-scaling rules based on CPU and memory thresholds.
* **NFR-SCAL-02 (Server-Side Pagination):** The system MUST NOT attempt to load entire SQL tables into memory. All data grid components MUST implement server-side pagination, fetching a maximum of 100 rows per request by default.
* **NFR-SCAL-03 (Connection Pooling):** The Data Access Layer MUST utilize connection pooling to manage concurrent SQL connections efficiently and prevent port exhaustion on the LOB database.

## 3. Availability & Reliability
The solution must provide enterprise-grade uptime for the configuration and routing layers.

### 3.1. Uptime & Resilience
* **NFR-AVAIL-01 (SaaS Control Plane SLA):** The SaaS Metadata Engine and API Gateway MUST support an availability SLA of **99.9%** (excluding customer-owned SQL database downtime).
* **NFR-AVAIL-02 (Multi-Region Deployment):** The backend infrastructure MUST be deployed in an Active/Passive or Active/Active multi-region configuration to survive regional cloud outages.
* **NFR-AVAIL-03 (Graceful Degradation):** If the customer's LOB SQL database is unreachable or times out, the Teams UI MUST NOT crash. It must display a user-friendly error message indicating the specific data source is temporarily unavailable.

## 4. Observability & Manageability
Comprehensive logging is required to support IT audits and operational troubleshooting.

### 4.1. Logging & Auditing
* **NFR-OBS-01 (Audit Trail):** The API MUST log all execution attempts (who, what, when). Logged data must include the user's UPN, the Stored Procedure/View name, and timestamp. **LOB data payloads and parameters must NOT be logged** to prevent PII/PHI leakage.
* **NFR-OBS-02 (Distributed Tracing):** Every request initiated from the Teams client MUST generate a unique Correlation ID, which is passed through the API Gateway and logged, allowing end-to-end trace tracking in Azure Application Insights.

## 5. Compatibility & Portability

### 5.1. Client & Platform Support
* **NFR-COMP-01 (Teams Cross-Platform):** The application UI MUST be fully responsive and functional across the Microsoft Teams Desktop Client (Windows/Mac), Teams on the Web, and Teams Mobile (iOS/Android).
* **NFR-COMP-02 (SQL Versioning):** The data connector MUST support Microsoft SQL Server 2016 and newer, Azure SQL Database, and Azure SQL Managed Instance.
