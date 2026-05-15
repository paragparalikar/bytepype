# Bytepype

A distributed, multi-tenant **Change Data Capture (CDC)** platform for building real-time data pipelines across heterogeneous data sources.

---

## 📖 What is Bytepype?

Bytepype is an open-source, self-service data pipeline platform that enables teams to visually configure, manage, and monitor real-time data flows between distributed systems — databases, message brokers, APIs, and search engines — without writing integration code.

Built with a security-first, multi-tenant architecture, Bytepype supports:
- Federated identity across multiple OAuth2/OIDC providers
- Per-tenant role-based access control
- API rate limiting out of the box

---

## ✨ Key Highlights
- **Distributed CDC Engine** — Oracle LogMiner-based redo log parsing with SCN-range tracking  
- **Multi-Tenant by Design** — Tenant-isolated connectors, dataflows, and audit trails with per-user RBAC  
- **Federated Identity** — Google and Microsoft Entra ID support with dynamic issuer resolution  
- **Pluggable Connector Architecture** — Oracle, Kafka, HTTP, Elasticsearch  
- **Single-Artifact Deployment** — Angular SPA embedded in Spring Boot JAR  
- **Production-Grade Security** — Rate limiting, CORS, JWT validation, entity-level caching  

---

## 🏗 System Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        UI["Angular 19 SPA<br/>(TailwindCSS + PrimeNG)"]
        OIDC_G["Google OIDC"]
        OIDC_M["Microsoft Entra ID"]
    end

    subgraph "API Gateway & Security Layer"
        SEC["Security Filter Chain<br/>(OAuth2 Resource Server)"]
        RL["Rate Limiter<br/>(Bucket4j + Ehcache)"]
        JWT["Dynamic JWT Issuer<br/>Resolution"]
        CORS["CORS Policy"]
    end

    subgraph "Application Core (bytepype-core)"
        API_C["Connector REST API"]
        API_D["Dataflow REST API"]
        API_U["User & Role REST API"]
        SVC["Service Layer<br/>(Validation + Business Logic)"]
        MAP["MapStruct DTO Mappers<br/>(Subclass-aware)"]
        AUDIT["Audit Infrastructure<br/>(CreatedBy / ModifiedBy)"]
    end

    subgraph "CDC Engine (bytepype-api)"
        REDO["Redo Log Parser"]
        SCN["SCN Range Tracker"]
        OPS["DML Operation Classifier"]
    end

    subgraph "Data Sources & Sinks"
        ORA[("Oracle DB")]
        KAFKA[("Kafka Cluster")]
        HTTP_EXT["HTTP / REST APIs"]
        ES[("Elasticsearch")]
    end

    subgraph "Persistence & Caching"
        JPA["Spring Data JPA<br/>(Hibernate)"]
        CACHE["Ehcache (JSR-107)<br/>Entity + Token Cache"]
        DB[("Relational DB")]
    end

    UI -->|"JWT Bearer Token"| SEC
    OIDC_G -->|"id_token"| UI
    OIDC_M -->|"authorization_code"| UI
    SEC --> RL
    SEC --> JWT
    SEC --> CORS
    JWT -->|"Trusted Issuer Check"| SVC
    RL -->|"Per-IP / Per-User Throttle"| SVC
    SVC --> API_C
    SVC --> API_D
    SVC --> API_U
    API_C --> MAP
    API_D --> MAP
    MAP --> JPA
    SVC --> AUDIT
    JPA --> CACHE
    JPA --> DB
    API_C -->|"Connector Config"| ORA
    API_C -->|"Connector Config"| KAFKA
    API_C -->|"Connector Config"| HTTP_EXT
    API_C -->|"Connector Config"| ES
    REDO --> ORA
    REDO --> SCN
    REDO --> OPS

🔒 Security Architecture

sequenceDiagram
    participant User as End User
    participant SPA as Angular SPA
    participant IDP as Identity Provider<br/>(Google / Microsoft)
    participant GW as Security Filter Chain
    participant JR as JWT Issuer Resolver
    participant UC as User JWT Converter
    participant US as User Service
    participant DB as Database

    User->>SPA: Access Application
    SPA->>IDP: Redirect to OIDC Login
    IDP->>SPA: Return JWT (id_token / code)
    SPA->>GW: API Request + Bearer Token

    GW->>GW: Bucket4j Rate Limit Check
    Note over GW: 30 req/min per IP

    GW->>JR: Resolve Issuer from JWT
    JR->>JR: Check Trusted Issuers Set
    JR->>JR: Lazy-init AuthenticationManager<br/>(ConcurrentHashMap cache)
    JR->>UC: Convert JWT to UserAuthenticationToken
    UC->>US: Find User by Email
    
    alt User Exists
        UC->>US: Update Profile from JWT Claims
    else New User
        UC->>US: Auto-Provision User
    end
    
    US->>DB: Persist User
    UC->>GW: Return UserAuthenticationToken
    GW->>SPA: Authenticated Response

🧩 Multi-Tenant Data Pipeline Model

erDiagram
    USER ||--o{ ROLE : "has many"
    ROLE ||--o{ AUTHORITY : "grants"
    USER ||--o{ CONNECTOR : "creates"
    USER ||--o{ DATAFLOW : "creates"
    CONNECTOR ||--o{ DATAFLOW : "source of"
    CONNECTOR ||--o{ DATAFLOW : "destination of"

⚙️ Connector Inheritance & Extensibility

classDiagram
    class AbstractAuditable {
        <<abstract>>
        -String createdBy
        -Instant createdDate
        -String lastModifiedBy
        -Instant lastModifiedDate
    }

    class Connector {
        <<abstract>>
        -Long id
        -String name
        -String description
        -ConnectorType type
        +Object get()
    }

    class OracleConnector {
        -String url
        -String username
        -String password
        +DataSource get()
    }

    class KafkaConnector {
        +Producer get()
    }

    class HttpConnector {
        +HttpClient get()
    }

    class ElasticsearchConnector {
        +RestClient get()
    }

    class ConnectorType {
        <<enumeration>>
        String ORACLE
        String KAFKA
        String HTTP
        String LOG
        String ELASTICSEARCH
    }

    AbstractAuditable <|-- Connector
    Connector <|-- OracleConnector
    Connector <|-- KafkaConnector
    Connector <|-- HttpConnector
    Connector <|-- ElasticsearchConnector
    Connector --> ConnectorType

🔄 CDC Engine — Oracle LogMiner Integration

graph LR
    subgraph "Oracle Database"
        REDO_LOG["Redo Log Files"]
        LOGMINER["LogMiner API"]
    end

    subgraph "Bytepype CDC Engine"
        PARSER["RedoLog Parser"]
        SCN_TRACK["SCN Range Tracker"]
        OP_CLASS["Operation Classifier"]
        MAPPER["RedoLog Mapper"]
    end

    subgraph "Downstream Sinks"
        KAFKA_SINK["Kafka Topic"]
        HTTP_SINK["REST API"]
        ES_SINK["Elasticsearch Index"]
    end

    REDO_LOG --> LOGMINER
    LOGMINER -->|"SQL_REDO + SCN"| PARSER
    PARSER --> SCN_TRACK
    PARSER --> OP_CLASS
    OP_CLASS -->|"Filter DML Only"| MAPPER

    MAPPER --> KAFKA_SINK
    MAPPER --> HTTP_SINK
    MAPPER --> ES_SINK

📂 Project Structure

bytepype/
├── pom.xml
├── bytepype-api/        # CDC domain model (no Spring deps)
└── bytepype-core/       # Full-stack application module

🛠 Tech Stack

Language: Java 21

Framework: Spring Boot 3.5.5

Security: Spring Security + OAuth2 Resource Server

Persistence: Spring Data JPA + Hibernate

Caching: Ehcache 3.11

Rate Limiting: Bucket4j

DTO Mapping: MapStruct

API Docs: SpringDoc OpenAPI (Swagger UI)

Frontend: Angular 19 + Tailwind CSS

Auth Client: angular-auth-oidc-client

Build: Maven multi-module

Monitoring: Spring Actuator

🚀 Quick Start

# Clone the repository
git clone https://github.com/<your-username>/bytepype.git
cd bytepype

# Build the project
mvn clean install

# Run the application
cd bytepype-core
mvn spring-boot:run

Application starts at: http://localhost:8080

📜 API Reference

Swagger UI available at /swagger-ui/index.html.

Core Endpoints:

GET /api/connectors — List connectors

POST /api/connectors — Create connector

`PUT /api
