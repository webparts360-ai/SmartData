Here are the visual architecture diagrams for the **Teams-Based Data Explorer**, following the **C4 Model** (Context, Container, Component). Here are the visual architecture diagrams for the **Teams-Based SQL Data Explorer**, following the **C4 Model** (Context, Container, Component). 

---

### Level 1: System Context Diagram
**Purpose:** Shows the high-level system, the users interacting with it, and the external systems it integrates with. It focuses on the "big picture."

```mermaid
graph TD
    %% Entities
    User([Business User / Operations])
    Admin([IT / Data Admin])
    
    subgraph External Systems
        Teams[Microsoft Teams]
        Entra[Microsoft Entra ID]
        SQL[(Customer SQL Database \n LOB Data)]
    end

    System{Teams-Based SQL \nData Explorer}

    %% Relationships
    User -->|Explores data, navigates relations| Teams
    Admin -->|Configures views, SPs, & access| System
    Teams -->|Hosts App Tab & passes SSO context| System
    System -.->|Authenticates & validates groups| Entra
    System ===>|Reads Views & executes Stored Procedures| SQL
    
    classDef system fill:#0078d4,stroke:#005a9e,stroke-width:2px,color:#fff;
    class System system;
```

---

### Level 2: Container Diagram
**Purpose:** Zooms into the System to show the high-level executable containers (applications, APIs, databases) and how they communicate.

```mermaid
graph TD
    User([Business User]) -->|Interacts via| TeamsApp

    subgraph Data Explorer SaaS Environment
        TeamsApp[Frontend SPA \n React / Fluent UI]
        API[API Gateway & App Service \n Handles logic & security]
        MetaDB[(Metadata Config Store \n Maps SQL to UI)]
    end

    Entra[Microsoft Entra ID]
    SQL[(Customer LOB SQL DB)]

    %% Relationships
    TeamsApp -->|REST/HTTPS \n Sends UI events & SSO Token| API
    API -->|Validates JWT Token & Group Claims| Entra
    API <-->|Reads admin configs, PK/FK rules| MetaDB
    API ===>|Executes parameterized SQL/SPs \n Applies Row-Level Security| SQL
    
    classDef container fill:#107c10,stroke:#0b5a0b,stroke-width:2px,color:#fff;
    class TeamsApp,API,MetaDB container;
```

---

### Level 3: Component Diagram (API / Backend Layer)
**Purpose:** Zooms into the **API Gateway & App Service** container to show the internal modular components that make the dynamic generation possible.

```mermaid
graph TD
    SPA[Frontend SPA] -->|API Requests| Auth

    subgraph API Gateway & App Service
        Auth[Auth & Context Manager \n Extracts user identity & roles]
        MetaEngine[Metadata Mapper Engine \n Translates UI requests to SQL maps]
        QueryGateway[Query Execution Gateway \n Safely parameterizes SP calls]
        RBAC[Granular Authorization Engine \n Checks Entity/Action permissions]
        DAL[Data Access Layer \n Manages SQL connections]
    end

    MetaDB[(Metadata Config Store)]
    SQL[(Customer LOB SQL DB)]

    %% Internal Flow
    Auth --> RBAC
    RBAC -->|If Authorized| MetaEngine
    MetaEngine <-->|Fetch View/SP mapping| MetaDB
    MetaEngine -->|Passes schema context| QueryGateway
    QueryGateway -->|Prepares secure payload| DAL
    DAL ===>|Executes live query| SQL
    
    classDef component fill:#6264a7,stroke:#464775,stroke-width:2px,color:#fff;
    class Auth,MetaEngine,QueryGateway,RBAC,DAL component;
```

---

### Notes on Level 4 (Code)
In the C4 model, Level 4 (Code) is typically omitted unless documenting a highly complex, non-standard algorithm. Because this solution relies on standard mapping patterns (translating JSON metadata into SQL parameters and UI components), the Component layer provides sufficient detail for architects and engineering leads to begin implementation.

These are generated using Mermaid.js, which renders directly in most modern Markdown viewers.

---

### Level 1: System Context Diagram
**Purpose:** Shows the high-level system, the users interacting with it, and the external systems it integrates with. It focuses on the "big picture."

```mermaid
graph TD
    %% Entities
    User([Business User / Operations])
    Admin([IT / Data Admin])
    
    subgraph External Systems
        Teams[Microsoft Teams]
        Entra[Microsoft Entra ID]
        SQL[(Customer SQL Database \n LOB Data)]
    end

    System{Teams-Based SQL \nData Explorer}

    %% Relationships
    User -->|Explores data, navigates relations| Teams
    Admin -->|Configures views, SPs, & access| System
    Teams -->|Hosts App Tab & passes SSO context| System
    System -.->|Authenticates & validates groups| Entra
    System ===>|Reads Views & executes Stored Procedures| SQL
    
    classDef system fill:#0078d4,stroke:#005a9e,stroke-width:2px,color:#fff;
    class System system;
```

---

### Level 2: Container Diagram
**Purpose:** Zooms into the System to show the high-level executable containers (applications, APIs, databases) and how they communicate.

```mermaid
graph TD
    User([Business User]) -->|Interacts via| TeamsApp

    subgraph Data Explorer SaaS Environment
        TeamsApp[Frontend SPA \n React / Fluent UI]
        API[API Gateway & App Service \n Handles logic & security]
        MetaDB[(Metadata Config Store \n Maps SQL to UI)]
    end

    Entra[Microsoft Entra ID]
    SQL[(Customer LOB SQL DB)]

    %% Relationships
    TeamsApp -->|REST/HTTPS \n Sends UI events & SSO Token| API
    API -->|Validates JWT Token & Group Claims| Entra
    API <-->|Reads admin configs, PK/FK rules| MetaDB
    API ===>|Executes parameterized SQL/SPs \n Applies Row-Level Security| SQL
    
    classDef container fill:#107c10,stroke:#0b5a0b,stroke-width:2px,color:#fff;
    class TeamsApp,API,MetaDB container;
```

---

### Level 3: Component Diagram (API / Backend Layer)
**Purpose:** Zooms into the **API Gateway & App Service** container to show the internal modular components that make the dynamic generation possible.

```mermaid
graph TD
    SPA[Frontend SPA] -->|API Requests| Auth

    subgraph API Gateway & App Service
        Auth[Auth & Context Manager \n Extracts user identity & roles]
        MetaEngine[Metadata Mapper Engine \n Translates UI requests to SQL maps]
        QueryGateway[Query Execution Gateway \n Safely parameterizes SP calls]
        RBAC[Granular Authorization Engine \n Checks Entity/Action permissions]
        DAL[Data Access Layer \n Manages SQL connections]
    end

    MetaDB[(Metadata Config Store)]
    SQL[(Customer LOB SQL DB)]

    %% Internal Flow
    Auth --> RBAC
    RBAC -->|If Authorized| MetaEngine
    MetaEngine <-->|Fetch View/SP mapping| MetaDB
    MetaEngine -->|Passes schema context| QueryGateway
    QueryGateway -->|Prepares secure payload| DAL
    DAL ===>|Executes live query| SQL
    
    classDef component fill:#6264a7,stroke:#464775,stroke-width:2px,color:#fff;
    class Auth,MetaEngine,QueryGateway,RBAC,DAL component;
```

---

### Notes on Level 4 (Code)
In the C4 model, Level 4 (Code) is typically omitted unless documenting a highly complex, non-standard algorithm. Because this solution relies on standard mapping patterns (translating JSON metadata into SQL parameters and UI components), the Component layer provides sufficient detail for architects and engineering leads to begin implementation.
