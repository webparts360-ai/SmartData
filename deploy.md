Here is the **Deployment & Infrastructure as Code (IaC) Blueprint** for SmartData - the Teams-Based Data Explorer app. This plan outlines the Azure-native infrastructure required to host the SaaS control plane and the automated deployment strategy using Infrastructure as Code (e.g., Terraform or Azure Bicep).

---

# Deployment & Infrastructure as Code (IaC) Blueprint

## 1. Deployment Strategy Overview
The solution uses a **Multi-Tenant SaaS deployment model** hosted on Microsoft Azure. 
* **Control Plane (SaaS):** Hosts the SPA, API Gateway, and Metadata Database. Scaled globally.
* **Data Plane (Customer LOB):** Remains in the customer’s environment. Connected dynamically at runtime.
* **IaC Tooling:** Azure Bicep or HashiCorp Terraform to ensure idempotent, repeatable, and version-controlled infrastructure provisioning.

## 2. Target Azure Architecture (PaaS Stack)
The architecture relies entirely on Azure Platform-as-a-Service (PaaS) offerings to minimize maintenance overhead and maximize auto-scaling capabilities.

* **Frontend Hosting:** **Azure Static Web Apps** (Hosts the React/Fluent UI SPA loaded inside the Teams iframe).
* **API Gateway & Logic:** **Azure App Service (Linux)** or **Azure Container Apps** (Stateless compute executing the metadata mapping and query translation).
* **Metadata Store:** **Azure SQL Database** (Serverless tier to manage tenant configurations, view mappings, and relationships cost-effectively).
* **Caching Layer:** **Azure Cache for Redis** (Caches configuration payloads and Entra ID group resolutions to achieve <200ms API latency).
* **Secret Management:** **Azure Key Vault** (Stores customer LOB database connection strings, API keys, and certificate thumbprints).
* **Observability:** **Azure Application Insights & Log Analytics** (Distributed tracing, audit logging, and performance monitoring).

## 3. IaC Module Breakdown (The Blueprint)
The IaC repository should be structured into modular components, allowing isolated updates and environment promotion (Dev, Test, Prod).

### Module 1: Foundation & Networking (`network.bicep`)
* Provisions the Azure Virtual Network (VNet) and Subnets.
* Configures **Azure Private Link / Private Endpoints** to ensure the API Gateway can securely route traffic to Key Vault, Redis, and Customer SQL databases without traversing the public internet.

### Module 2: Data & State (`data.bicep`)
* Provisions the Azure SQL Database (Metadata Store) with automated backups and Entra ID-only authentication enabled.
* Provisions Azure Cache for Redis.

### Module 3: Security & Identity (`security.bicep`)
* Provisions Azure Key Vault with RBAC access policies granting "Secrets User" role strictly to the API Gateway's Managed Identity.
* Registers the **Microsoft Entra ID Application** (App Registration) required for Teams SSO, defining the necessary API scopes (`access_as_user`).

### Module 4: Compute & Web (`compute.bicep`)
* Provisions the Azure App Service Plan (Standard/Premium tier for VNet integration).
* Deploys the App Service for the API Gateway, configuring Managed Identities and VNet integration for outbound traffic.
* Deploys the Azure Static Web App for the Teams SPA.

## 4. CI/CD Pipeline Integration (GitHub Actions / Azure DevOps)
Infrastructure and application code changes are deployed via automated pipelines.

1. **Continuous Integration (CI):**
   * Triggered on Pull Request to `main`.
   * Lints IaC code (e.g., `bicep lint` or `terraform validate`).
   * Builds the React SPA and .NET/Node.js API Gateway.
   * Runs unit tests (specifically testing the Query Gateway parameterization logic to prevent SQL injection).
2. **Continuous Deployment (CD):**
   * **Step 1:** Executes the IaC plan (`bicep deploy` or `terraform apply`) to provision or update Azure resources.
   * **Step 2:** Deploys the API Gateway code to Azure App Service via ZipDeploy or Container Registry.
   * **Step 3:** Deploys the compiled SPA to Azure Static Web Apps.
   * **Step 4:** Packages the Teams App Manifest (`manifest.zip`) and publishes it to the Microsoft Teams Admin Center for tenant approval.

## 5. Customer LOB Connectivity Patterns (Runtime)
While the IaC provisions the SaaS infrastructure, the system must connect to the customer's SQL database dynamically:

* **Pattern A: Cloud-to-Cloud (Azure SQL / Managed Instance):** The customer provisions a Private Endpoint in their Azure VNet, linking to the SaaS VNet, allowing the API Gateway to query their DB entirely over the Azure backbone.
* **Pattern B: Cloud-to-Ground (On-Premises SQL):** The customer installs the **Microsoft On-Premises Data Gateway** inside their firewall. The SaaS API routes queries through Azure Relay to the on-premise agent, requiring zero inbound open firewall ports.
