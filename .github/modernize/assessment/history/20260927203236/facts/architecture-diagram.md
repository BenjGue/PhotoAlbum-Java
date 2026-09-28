# Architecture Diagram

PhotoAlbum-Java is a single-module Spring Boot web application that provides a server-rendered photo gallery and JSON-based uploads. It uses Spring Security for endpoint protection and persists photo metadata and binary image data in Oracle Database.

## Application Architecture

```mermaid
flowchart TD
    subgraph Client["Client Layer"]
        Browser["Web Browser"]
        UploadJs["Upload JavaScript"]
    end
    subgraph App["Application Layer - Spring Boot 2.7"]
        MVC["Spring MVC Controllers"]
        Views["Thymeleaf Templates"]
        Security["Spring Security HTTP Basic"]
        Service["Transactional Photo Service"]
        Validation["Upload and Image Validation"]
    end
    subgraph Data["Data Layer"]
        JPA["Spring Data JPA and Hibernate"]
        Entity["Photo Entity"]
        Oracle[("Oracle Database Free 23ai")]
        Blob["PHOTOS table with BLOB image data"]
    end
    subgraph Runtime["Runtime and Delivery"]
        Jar["Executable Java 8 JAR"]
        Container["Docker Container"]
    end

    Browser -->|"HTTP and HTML requests"| MVC
    UploadJs -->|"multipart upload requests"| MVC
    MVC -->|"renders"| Views
    Security -.->|"protects upload and delete endpoints"| MVC
    MVC -->|"delegates photo operations"| Service
    Service -->|"validates files and dimensions"| Validation
    Service -->|"uses entities and queries"| JPA
    JPA -->|"JDBC and SQL"| Oracle
    Oracle -->|"stores metadata and image bytes"| Blob
    Container -->|"runs"| Jar
    Jar -->|"serves on port 8080"| MVC
```

### Technology Stack Summary

| Layer | Technology | Version | Purpose |
|---|---|---|---|
| Runtime | Java | 8 | Application runtime and compilation target |
| Build | Maven | 3.9.6 in Docker | Dependency resolution and packaging |
| Application | Spring Boot | 2.7.18 | Bootstrapping and embedded web runtime |
| Presentation | Spring MVC | Spring Boot managed | Routes HTML pages, JSON upload responses, and photo resources |
| Presentation | Thymeleaf | Spring Boot managed | Server-side rendering of gallery and detail pages |
| Security | Spring Security | Spring Boot managed | Stateless HTTP Basic authentication for state-changing endpoints |
| Business Logic | Spring services and transactions | Spring Boot managed | Photo upload, retrieval, navigation, and deletion workflows |
| Data Access | Spring Data JPA and Hibernate | Spring Boot managed | Repository abstraction and ORM persistence |
| Database | Oracle Database Free | 23ai container image | Stores photo metadata and image BLOBs |
| Database Driver | Oracle JDBC ojdbc8 | Maven managed | JDBC connectivity to Oracle |
| Packaging | Docker and Spring Boot JAR | Temurin 8 | Containerized application delivery |

### Data Storage & External Services

Oracle Database is the only runtime data store. The `PHOTOS` table contains photo metadata and the image bytes in a BLOB column; the compatibility file path is not used to serve images. The application has no external API, cache, message broker, or email integration. Docker Compose supplies the Oracle service and connects it to the application over the internal `photoalbum-network`; administrator and database credentials are injected through environment variables.

### Key Architectural Decisions

- Uses a conventional Spring MVC, service, and repository layering with constructor dependency injection.
- Stores uploaded image content directly in Oracle BLOB storage while retaining generated filename and path fields for compatibility.
- Uses Oracle-specific native SQL for ordering, navigation, pagination, month filtering, and analytic statistics.

## Component Relationships

```mermaid
flowchart LR
    subgraph Presentation["Presentation"]
        HomeCtrl["HomeController"]
        DetailCtrl["DetailController"]
        PhotoFileCtrl["PhotoFileController"]
        Templates["Thymeleaf Templates"]
    end
    subgraph Business["Business Logic"]
        PhotoSvc["PhotoService"]
        PhotoSvcImpl["PhotoServiceImpl"]
        UploadValidation["File and Image Validation"]
    end
    subgraph DataAccess["Data Access"]
        PhotoRepo["PhotoRepository"]
        PhotoEntity["Photo Entity"]
    end
    subgraph Infrastructure["Infrastructure"]
        SecurityConfig["SecurityConfig"]
        SecurityFilter["Security Filter Chain"]
        JPAProvider["JPA and Hibernate"]
    end

    HomeCtrl -->|"delegates gallery and upload"| PhotoSvc
    DetailCtrl -->|"delegates detail and delete"| PhotoSvc
    PhotoFileCtrl -->|"delegates binary retrieval"| PhotoSvc
    HomeCtrl -->|"renders index"| Templates
    DetailCtrl -->|"renders detail"| Templates
    PhotoSvc -.->|"implemented by"| PhotoSvcImpl
    PhotoSvcImpl -->|"validates uploads"| UploadValidation
    PhotoSvcImpl -->|"queries and persists"| PhotoRepo
    PhotoRepo -->|"maps"| PhotoEntity
    PhotoRepo -->|"uses"| JPAProvider
    SecurityConfig -->|"configures"| SecurityFilter
    SecurityFilter -.->|"intercepts requests"| HomeCtrl
    SecurityFilter -.->|"intercepts requests"| DetailCtrl
    JPAProvider -->|"persists"| PhotoEntity
```

### Component Inventory

| Component | Layer | Type | Responsibility |
|---|---|---|---|
| HomeController | Presentation | Spring MVC Controller | Renders the gallery and handles multipart photo uploads with JSON responses |
| DetailController | Presentation | Spring MVC Controller | Displays a photo, resolves adjacent-photo navigation, and handles deletion |
| PhotoFileController | Presentation | Spring MVC Controller | Reads stored image bytes and serves them as HTTP resources |
| Thymeleaf Templates | Presentation | Server-rendered Views | Renders the gallery layout, index page, and detail page |
| PhotoService | Business Logic | Service Interface | Defines photo retrieval, upload, navigation, and deletion operations |
| PhotoServiceImpl | Business Logic | Transactional Service | Validates files, extracts dimensions, and coordinates persistence workflows |
| File and Image Validation | Business Logic | Service Concern | Enforces MIME, size, non-empty file, and maximum pixel-count constraints |
| PhotoRepository | Data Access | Spring Data JPA Repository | Executes CRUD operations and Oracle-native photo queries |
| Photo Entity | Data Access | JPA Entity | Maps photo metadata and BLOB content to the `PHOTOS` table |
| SecurityConfig | Infrastructure | Spring Configuration | Defines password encoding, in-memory admin identity, and endpoint authorization |
| Security Filter Chain | Infrastructure | Spring Security Filter | Applies stateless HTTP Basic authentication to upload and delete requests |
| JPA and Hibernate | Infrastructure | Persistence Provider | Converts repository operations into JDBC and SQL database access |
