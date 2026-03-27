**Integration & API Specification** for the SmartData - Teams-Based Data Explorer. This document defines the contract between the Microsoft Teams frontend (SPA), the SaaS API Gateway, and the customer’s Line-of-Business (LOB) SQL databases.

---

# Integration & API Specification

## 1. Integration Architecture & Authentication Flow

The solution relies on a delegated trust model using **Microsoft Entra ID (SSO)** and **SQL Server Session Context** to ensure end-to-end security without middle-tier data storage.

### 1.1. The Authentication Handshake
1. **Teams Client:** The React SPA calls `microsoftTeams.authentication.getAuthToken()` to silently acquire an Entra ID JWT token for the current user.
2. **API Gateway:** Receives the token in the `Authorization: Bearer <token>` header. It validates the token signature, audience, and extracts the `oid` (User Object ID) and `groups` claims.
3. **Database Execution:** The API Gateway opens a connection to the customer's SQL database and executes `sp_set_session_context` to pass the user's `oid` directly to the SQL engine before executing any View or Stored Procedure.

### 1.2. SQL Row-Level Security (RLS) Integration
To enforce data security at the database tier, the customer's SQL environment utilizes the injected session context:
```sql
-- Executed by the API Gateway immediately upon opening a connection
EXEC sp_set_session_context @key = N'UserId', @value = 'user-entra-object-id';

-- Customer's SQL RLS Security Predicate relies on this context
CREATE FUNCTION Security.fn_UserAccessPredicate(@AssignedUserId UNIQUEIDENTIFIER)
RETURNS TABLE
AS
RETURN SELECT 1 AS fn_accessResult 
WHERE @AssignedUserId = TRY_CAST(SESSION_CONTEXT(N'UserId') AS UNIQUEIDENTIFIER);
```

---

## 2. Core API Specification (RESTful JSON)

The API acts as a "Backend-for-Frontend" (BFF), serving metadata to generate the UI and acting as a secure proxy to the LOB database. All endpoints require a valid Entra ID Bearer token.

### 2.1. UI Metadata Endpoint
Retrieves the configuration blueprint to dynamically render the Teams application. It filters out any Entities, Fields, or Actions the user does not have Entra Group permissions to see.

*   **Endpoint:** `GET /api/v1/apps/{appId}/metadata`
*   **Response (Application/JSON):**
    ```json
    {
      "appId": "123e4567-e89b-12d3-a456-426614174000",
      "appName": "Customer Operations Portal",
      "entities": [
        {
          "entityId": "abc-123",
          "displayName": "Customers",
          "primaryKeyColumn": "CustomerID",
          "fields": [
            { "columnName": "Name", "displayName": "Company Name", "dataType": "String", "isVisibleInList": true }
          ],
          "actions": [
            {
              "actionId": "act-789",
              "displayName": "Search by Region",
              "type": "Search",
              "parameters": [
                { "parameterName": "@Region", "uiControlType": "ComboBox", "isRequired": true }
              ]
            }
          ]
        }
      ]
    }
    ```

### 2.2. Data Query Endpoint (SQL Views)
Executes a read-only query against the mapped SQL View. Supports server-side pagination, sorting, and basic filtering.

*   **Endpoint:** `POST /api/v1/apps/{appId}/entities/{entityId}/query`
*   **Request Body:**
    ```json
    {
      "pageNumber": 1,
      "pageSize": 50,
      "sortBy": "CreatedAt",
      "sortDirection": "DESC",
      "filters": [
        { "columnName": "Status", "operator": "Equals", "value": "Active" }
      ]
    }
    ```
*   **Response:** Returns a flat JSON array representing the rows retrieved from the SQL View, mapped exactly to the columns defined in the metadata.

### 2.3. Relational Drill-Down Endpoint
Fetches related records when a user clicks into a specific row (e.g., clicking a Customer to see their Orders), utilizing the `Relationship` metadata configurations.

*   **Endpoint:** `GET /api/v1/apps/{appId}/entities/{targetEntityId}/related/{sourceEntityId}/{pkValue}`
*   **Description:** The API looks up the configured Foreign Key mapping between `sourceEntityId` and `targetEntityId`, dynamically generating a `WHERE TargetColumn = @pkValue` SQL query against the target view.

### 2.4. Action Execution Endpoint (Stored Procedures)
Translates the dynamic Teams UI form submission into a parameterized SQL Stored Procedure execution.

*   **Endpoint:** `POST /api/v1/apps/{appId}/actions/{actionId}/execute`
*   **Request Body:**
    ```json
    {
      "parameters": {
        "@Region": "North America",
        "@StartDate": "2026-01-01T00:00:00Z"
      }
    }
    ```
*   **Behavior:** 
    1. Validates the incoming parameters against the `ActionParameter` metadata schema.
    2. Opens SQL Connection & sets `SESSION_CONTEXT`.
    3. Executes the stored procedure safely using `SqlCommand` with `SqlParameter` objects to prevent SQL Injection.
    4. **Response:** Returns either a dataset (if `ActionType == Search`) or a success/failure status (if `ActionType == Execute`).

---

## 3. Network & Connectivity Integration

To connect the SaaS API Gateway to the customer's LOB SQL database, the solution supports two enterprise integration patterns:

1. **Cloud-Native (Azure SQL / Managed Instance):** 
   Integration is established via **Azure Private Link / Private Endpoint**. This ensures SQL traffic never traverses the public internet, routing directly through the Azure backbone.
2. **On-Premises SQL Server:**
   Integration utilizes the **On-Premises Data Gateway** (or an Azure Hybrid Connection), allowing the cloud API to securely query firewalled, on-premises SQL databases without requiring inbound firewall ports to be opened.
