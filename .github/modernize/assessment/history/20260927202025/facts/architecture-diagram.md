# Architecture Diagram

Photo Album is a single-module Spring Boot web application that serves a server-rendered photo gallery, accepts image uploads, and stores photo metadata and binary content in Oracle Database. The application is packaged as a Java 8 container and can run with an Oracle database through Docker Compose.

## Application Architecture

```mermaid
flowchart TD
    subgraph Client["Client Layer"]
        Browser["Web Browser"]
    end
    subgraph App["Application Layer - Spring Boot 2.7.18"]
        MVC["Spring MVC"]
        Views["Thymeleaf templates"]
        Controllers["Photo gallery controllers"]
        Security["Spring Security HTTP Basic"]
        Service["Photo service and upload validation"]
        Image["Java ImageIO processing"]
    end
    subgraph Data["Data Layer"]
        JPA["Spring Data JPA and Hibernate"]
        Entity["Photo entity"]
        DB[("Oracle Database Free 23ai")]
        Blob[("Oracle PHOTOS table BLOB")]
    end
    subgraph Runtime["Runtime and Operations"]
        Container["Eclipse Temurin 8 container"]
        Config["Environment-based configuration"]
    end

    Browser -->|"HTTP gallery, upload, and photo requests"| MVC
    MVC -->|"routes requests"| Security
    Security -->|"protects upload and delete endpoints"| Controllers
    Controllers -->|"renders pages and returns JSON or image bytes"| Views
    Controllers -->|"delegates photo operations"| Service
    Service -->|"reads image headers and validates uploads"| Image
    Service -->|"persists and queries photos"| JPA
    JPA -->|"maps entities and executes SQL"| Entity
    Entity -->|"CRUD and native Oracle queries"| DB
    DB -->|"stores metadata and binary content"| Blob
    Container -->|"runs application on port 8080"| MVC
    Config -->|"injects database and upload settings"| Service
    Config -->|"configures data source and security credentials"| Security
```

### Technology Stack Summary

| Layer | Technology | Version | Purpose |
|---|---|---|---|
| Runtime | Java | 8 | Application runtime and compilation target |
| Application | Spring Boot | 2.7.18 | Application bootstrap and auto-configuration |
| Presentation | Spring MVC | 5.3.x via Spring Boot | HTTP routing and controller handling |
| Presentation | Thymeleaf | Spring Boot managed | Server-side gallery and detail page rendering |
| Security | Spring Security | 5.7.x via Spring Boot | Stateless HTTP Basic authentication for state-changing endpoints |
| Business Logic | Spring service layer | Project code | Upload validation, image metadata extraction, navigation, and lifecycle operations |
| Data Access | Spring Data JPA and Hibernate | Spring Boot managed | Entity persistence and repository abstraction |
| Database Driver | Oracle JDBC ojdbc8 | Maven managed | JDBC connectivity to Oracle |
| Data Storage | Oracle Database Free 23ai | Docker image latest | Photo metadata and BLOB content storage |
| Containerization | Maven and Eclipse Temurin Docker images | Maven 3.9.6, Java 8 | Multi-stage build and application runtime |
| Validation | Spring Validation and ImageIO | Spring Boot managed, JDK 8 | Request constraints, MIME and size checks, and image dimension checks |

### Data Storage & External Services

Oracle Database is the only external service dependency. The `Photo` entity stores metadata and the image bytes in the `PHOTOS` table, with Oracle-specific native SQL for ordering, navigation, pagination, month filtering, and statistics. Docker Compose runs Oracle Database Free with a named data volume and connects the application over a private bridge network. There is no cache, message broker, remote API, or object-storage integration; the file path fields are retained for compatibility while photo serving reads BLOB data from Oracle.

### Key Architectural Decisions

- Uses a layered Spring MVC, service, and repository design with constructor injection between controllers, services, and data access.
- Stores photo binaries directly in Oracle BLOB columns rather than using a filesystem or object store.
- Protects upload and delete operations with stateless HTTP Basic authentication while leaving read-only gallery operations public.

## Component Relationships

```mermaid
flowchart LR
    subgraph Presentation["Presentation"]
        HomeCtrl["HomeController"]
        DetailCtrl["DetailController"]
        FileCtrl["PhotoFileController"]
        Templates["Thymeleaf templates"]
    end
    subgraph Business["Business Logic"]
        PhotoSvc["PhotoService"]
        PhotoSvcImpl["PhotoServiceImpl"]
        UploadResult["UploadResult"]
        ImageIO["ImageIO validation"]
    end
    subgraph DataAccess["Data Access"]
        PhotoRepo["PhotoRepository"]
        PhotoEntity["Photo entity"]
    end
    subgraph Infrastructure["Infrastructure"]
        SecurityCfg["SecurityConfig"]
        SecurityChain["Security filter chain"]
        OracleDB[("Oracle PHOTOS table")]
    end

    HomeCtrl -->|"delegates listing and uploads"| PhotoSvc
    DetailCtrl -->|"delegates detail and deletion"| PhotoSvc
    FileCtrl -->|"delegates binary retrieval"| PhotoSvc
    HomeCtrl -->|"renders index and upload JSON"| Templates
    DetailCtrl -->|"renders detail and redirects"| Templates
    PhotoSvc -.->|"implemented by"| PhotoSvcImpl
    PhotoSvcImpl -->|"returns upload outcome"| UploadResult
    PhotoSvcImpl -->|"checks type, size, and dimensions"| ImageIO
    PhotoSvcImpl -->|"queries and saves"| PhotoRepo
    PhotoRepo -->|"maps"| PhotoEntity
    PhotoRepo -->|"executes native SQL and CRUD"| OracleDB
    SecurityCfg -->|"creates"| SecurityChain
    SecurityChain -.->|"intercepts requests"| HomeCtrl
    SecurityChain -.->|"intercepts requests"| DetailCtrl
    SecurityChain -.->|"intercepts requests"| FileCtrl
```

### Component Inventory

| Component | Layer | Type | Responsibility |
|---|---|---|---|
| HomeController | Presentation | Spring MVC controller | Displays the gallery and handles multi-file upload requests |
| DetailController | Presentation | Spring MVC controller | Displays a photo, computes adjacent-photo navigation, and handles deletion |
| PhotoFileController | Presentation | Spring MVC controller | Reads photo bytes and serves them with the stored MIME type |
| Thymeleaf templates | Presentation | Server-rendered views | Renders the gallery, photo detail, and shared layout pages |
| PhotoService | Business Logic | Service interface | Defines photo listing, retrieval, upload, deletion, and navigation operations |
| PhotoServiceImpl | Business Logic | Spring service | Validates uploads, extracts dimensions, creates entities, and coordinates persistence |
| UploadResult | Business Logic | Result model | Carries upload success, file name, photo ID, and error details |
| ImageIO validation | Business Logic | JDK image processing | Inspects image headers and enforces the maximum pixel limit |
| PhotoRepository | Data Access | Spring Data JPA repository | Provides CRUD operations and Oracle native queries for photo retrieval |
| Photo | Data Access | JPA entity | Maps photo metadata and BLOB content to the `PHOTOS` table |
| SecurityConfig | Infrastructure | Spring configuration | Defines password encoding, in-memory admin identity, and endpoint authorization |
| Security filter chain | Infrastructure | Spring Security filter | Applies stateless HTTP Basic authentication to protected requests |
| Oracle PHOTOS table | Infrastructure | Relational database storage | Stores photo metadata and binary image content |
