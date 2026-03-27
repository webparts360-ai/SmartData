# Solution Architecture Document: Teams-Based Data Explorer

## 1. Executive Summary
SmartData is a Teams-based Data Explorer is a configuration-driven, no-code application embedded natively within Microsoft Teams. It bridges the gap between read-only reporting (Power BI) and custom development (Power Apps) by dynamically generating interactive data exploration interfaces directly from underlying SQL Views and Stored Procedures. 

**Positioning:** "Turn your databases into usable apps in Teams—without building anything."

## 2. Architectural Principles
* **Metadata-Driven UI:** The presentation layer is entirely dynamically generated based on SQL metadata (schema, views, stored procedure parameters) and admin configuration.
* **Zero Data Replication:** Executes real-time queries against LOB SQL databases. No data ingestion, replication, or semantic model refreshing is required.
* **Native Context:** Seamlessly integrated into Microsoft Teams to eliminate context switching for business users.
* **Secure by Design:** Leverages Microsoft Entra ID (formerly Azure AD) for identity, passing context down to the database level to support SQL Row-Level Security (RLS).

## 3. High-Level Architecture (Logical View)

The solution consists of four primary tiers:

### A. Presentation Layer (Microsoft Teams Client)
* **Teams App Manifest:** Packages the application as a Personal Tab or Channel Tab within Teams.
* **Dynamic UI Engine (SPA):** A frontend framework (e.g., React/Fluent UI) that renders listings, detail views, and navigation components on the fly based on metadata payloads.
* **Auto-Form Generator:** Instantly translates stored procedure parameters (e.g., `@StartDate`, `@CustomerID`) into native UI filters, dropdowns, and search inputs.

### B. Application & API Layer (Cloud Service / Azure)
* **Metadata Mapper Engine:** Stores admin configurations mapping SQL Views to UI lists, and defining Primary Key (PK) / Foreign Key (FK) relationships for cross-entity drill-down.
* **Query Execution Gateway:** Translates user interactions from the UI into parameterized SQL queries and stored procedure executions. 
* **Session & State Manager:** Manages user context and pagination for large datasets.

### C. Data Layer (Customer SQL Environment)
* **Read Models (SQL Views):** Provide flattened, optimized datasets for UI list rendering.
* **Action & Search Models (Stored Procedures):** Handle complex server-side business logic, search filtering, and transactional data retrieval.
* **Relational Graph:** Native SQL relationships modeled in the application layer to allow intuitive navigation (e.g., Customer → Orders → Order Lines).

### D. Identity & Security Layer
* **Microsoft Entra ID:** Provides Single Sign-On (SSO) and group-based access control.
* **Granular Authorization Engine:** Evaluates Entra ID group claims against configured access rules at the Entity, Field, Relationship, and Action (Stored Procedure) levels.

## 4. Data Flow & Execution Sequence

1. **Initialization:** User opens the app in Teams. Teams passes an Entra ID SSO token to the Application API.
2. **Metadata Retrieval:** The API validates the user, checks permissions, and retrieves the configuration metadata for the user's role.
3. **UI Rendering:** The Dynamic UI Engine renders the main dashboard/listing using the mapped SQL Views.
4. **Parameterization & Query:** When a user searches or filters, the UI dynamically displays form fields matching the SQL Stored Procedure parameters. 
5. **Real-Time Execution:** Upon submission, the API executes the LOB Stored Procedure in real-time.
6. **Drill-Down:** Clicking a record triggers a contextual fetch of related entities using pre-configured PK/FK mappings, loading child views instantly.

## 5. Security & Compliance Architecture

The solution employs a defense-in-depth strategy, ensuring that data exposure is strictly controlled:
* **Authentication:** Seamless Microsoft Teams SSO via Entra ID. No separate credentials required.
* **Application-Level RBAC:** Admins define visibility and execution rights mapping Entra ID groups to specific SQL Views and Stored Procedures.
* **Field-Level Security:** Sensitive columns can be masked or hidden based on user role.
* **Database-Level Security (RLS):** The API Gateway can pass the authenticated user's context (e.g., UPN or Object ID) via `SESSION_CONTEXT` to the SQL Server, enforcing native SQL Row-Level Security policies.

## 6. Architectural Differentiators & Baseline Comparison

This architecture drastically reduces Total Cost of Ownership (TCO) compared to alternative Microsoft stack solutions:

| Solution Baseline | Architectural Limitations | Data Explorer Advantage |
| :--- | :--- | :--- |
| **Microsoft Power Apps** | Requires manual screen building, state management, and custom connector configuration. | **Zero build time.** UI is generated instantly from SQL metadata. |
| **Microsoft Power BI** | Built for analytical processing (OLAP); requires semantic models and scheduled refreshes. | **Operational & Real-Time.** Direct execution against relational OLTP schemas. |
| **Microsoft Dataverse** | Requires expensive migration of LOB data into the Dataverse ecosystem. | **Data stays in place.** Connects directly to existing SQL infrastructure. |
| **Microsoft Lists** | Lacks relational depth; cannot handle complex stored procedures or millions of rows. | **Deeply Relational.** Built for large-scale enterprise SQL architectures. |

## 7. Deployment Model
* **SaaS Hosting:** The Application API and Metadata Engine are hosted in a multi-tenant cloud environment (e.g., Azure App Service/AKS).
* **Connectivity:** Connects to customer LOB SQL databases via secure API endpoints, Azure Private Link, or On-Premises Data Gateways (if SQL is self-hosted).
* **Licensing:** Metered via SaaS subscription (tiered per user/month) validated through Entra ID integration.
