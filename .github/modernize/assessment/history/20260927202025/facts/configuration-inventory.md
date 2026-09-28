# Configuration & Externalized Settings Inventory

The application has configuration in Maven, Spring property files, Docker Compose, a `.env` template, Oracle initialization scripts, and Azure PowerShell provisioning scripts. Runtime secrets are intended to come from environment variables; no external config server, vault integration, or Kubernetes configuration manifests were found.

## Configuration Sources

| Source | Type | Path/Location | Notes |
|---|---|---|---|
| Maven project descriptor | Build configuration | `pom.xml` | Spring Boot parent, Java/Maven compiler settings, dependencies, and Spring Boot packaging plugin. |
| Default Spring configuration | Application properties | `src/main/resources/application.properties` | Port, encoding, Oracle datasource defaults, JPA, uploads, and logging. |
| Docker Spring configuration | Profile properties | `src/main/resources/application-docker.properties` | Docker datasource overrides, upload settings, port, and reduced application logging. |
| Test Spring configuration | Test properties | `src/test/resources/application-test.properties` | H2 in-memory datasource, `create-drop`, test upload path, and test admin credentials. |
| Compose environment | Container orchestration configuration | `docker-compose.yml` | Oracle Free database and application services, environment variables, ports, volume, network, healthcheck, and dependency condition. |
| Environment template | Secret/configuration template | `.env.example` | Documents required Oracle and application admin values. A real `.env` is expected to be git-ignored. |
| Container build/runtime | Dockerfile | `Dockerfile` | Maven/Eclipse Temurin 8 build and runtime images; `JAVA_OPTS` defaults. |
| Oracle initialization | Database startup scripts | `oracle-init/01-create-user.sql`, `02-verify-user.sql`, `create-user.sh`, `healthcheck.sql` | Schema grants and readiness/user setup; credentials are supplied by the database container environment. |
| Azure provisioning | Infrastructure/bootstrap script | `azure-setup.ps1` | Creates ACR, AKS, and Azure Database for PostgreSQL, then writes PostgreSQL/Azure values to `.env`; it is not an application runtime config file. |
| Azure cleanup | Infrastructure lifecycle script | `azure-reset.ps1` | Removes ACR images and the `photo-album` AKS namespace. |
| External config server | Not found | — | No Spring Cloud Config, Consul, App Configuration, or external Git configuration repository reference. |
| Secret store | Not found | — | No Key Vault, HashiCorp Vault, AWS Secrets Manager, encrypted properties, or sealed secrets reference. |
| Kubernetes configuration | Not found | — | No ConfigMap, Secret, Deployment, Helm chart, or readiness manifest was found. |

## Build Profiles

| Profile | Activation | Purpose | Key Dependencies/Plugins |
|---|---|---|---|
| Maven default | Automatic when Maven runs without `-P` | Builds the single `photo-album` JAR | Spring Boot parent `2.7.18`; `spring-boot-maven-plugin`; Java source/target 8. |
| Maven named profiles | None declared | No project-specific `dev`, `test`, `docker`, or cloud build profiles are defined | No conditional dependencies or plugins beyond the default build. |
| Docker build stage | Explicitly selected by `docker build`/Compose | Resolves dependencies and packages the application in a container | `maven:3.9.6-eclipse-temurin-8`; `mvn clean package -DskipTests`. |

## Runtime Profiles

| Profile | Activation Method | Config Files | Key Overrides |
|---|---|---|---|
| Default | Spring default when no active profile is supplied | `src/main/resources/application.properties` | Oracle URL defaults to `oracle-db:1521/FREEPDB1`; application loggers use DEBUG. |
| `docker` | `SPRING_PROFILES_ACTIVE=docker` in `docker-compose.yml` | `application-docker.properties` layered with default properties | Same Oracle URL/environment binding, port 8080, upload limits, application logging INFO, Spring Web WARN, Hibernate SQL DEBUG. |
| `test` | Test context/property selection; no global active-profile setting found | `src/test/resources/application-test.properties` | H2 in-memory database, `create-drop`, test upload path, and test-only admin credentials. |

No `@Profile` annotations or combined profile activation were found. The Docker profile is explicitly selected by Compose; the test configuration is separate test-classpath configuration rather than a production profile.

## Properties Inventory

### Application and server

| Property Key | Default | Profiles | Source |
|---|---|---|---|
| `server.port` | `8080` | Default, docker | `application.properties`, repeated in `application-docker.properties`. |
| `server.servlet.encoding.charset` | `UTF-8` | Default, docker | Spring property files. |
| `server.servlet.encoding.enabled` | `true` | Default, docker | Spring property files. |
| `server.servlet.encoding.force` | `true` | Default, docker | Spring property files. |

### Database and JPA

| Property Key | Default | Profiles | Source |
|---|---|---|---|
| `spring.datasource.url` | `jdbc:oracle:thin:@oracle-db:1521/FREEPDB1` | Default, docker | `${SPRING_DATASOURCE_URL:...}` in both property files; Compose supplies the Docker value. |
| `spring.datasource.username` | `photoalbum` | Default, docker | `${SPRING_DATASOURCE_USERNAME:photoalbum}`; Compose maps `APP_USER`. |
| `spring.datasource.password` | Required; no default | Default, docker | `${SPRING_DATASOURCE_PASSWORD}`; Compose maps `APP_USER_PASSWORD`. |
| `spring.datasource.driver-class-name` | `oracle.jdbc.OracleDriver` | Default, docker | Spring property files. |
| `spring.jpa.database-platform` | `org.hibernate.dialect.OracleDialect` | Default, docker | Spring property files. |
| `spring.jpa.hibernate.ddl-auto` | `create` | Default, docker | Spring property files. |
| `spring.jpa.show-sql` | `true` | Default, docker | Spring property files. |
| `spring.jpa.properties.hibernate.format_sql` | `true` | Default, docker | Spring property files. |
| `spring.jpa.database-platform` | `org.hibernate.dialect.H2Dialect` | Test | `application-test.properties`. |
| `spring.jpa.hibernate.ddl-auto` | `create-drop` | Test | `application-test.properties`. |
| `spring.jpa.show-sql` | `false` | Test | `application-test.properties`. |

The test datasource is `jdbc:h2:mem:testdb`, driver `org.h2.Driver`, username `sa`, and an empty password.

### Upload and application settings

| Property Key | Default | Profiles | Source |
|---|---|---|---|
| `spring.servlet.multipart.max-file-size` | `10MB` | Default, docker | Spring property files. |
| `spring.servlet.multipart.max-request-size` | `50MB` | Default, docker | Spring property files. |
| `app.file-upload.max-file-size-bytes` | `10485760` (10 MiB) | Default, docker, test | Spring property files. |
| `app.file-upload.allowed-mime-types` | `image/jpeg,image/png,image/gif,image/webp` | Default, docker, test | Spring property files. |
| `app.file-upload.max-files-per-upload` | `10` | Default, docker, test | Spring property files. |
| `app.file-upload.upload-path` | Not set in production | Test: `target/test-uploads` | Test property only; production photo data is configured in the application code/database path. |

### Logging and security

| Property Key | Default | Profiles | Source |
|---|---|---|---|
| `logging.level.com.photoalbum` | `DEBUG` | Default; `INFO` in docker | Spring property files. |
| `logging.level.org.springframework.web` | `DEBUG` | Default; `WARN` in docker | Spring property files. |
| `logging.level.org.hibernate.SQL` | Not set | Docker: `DEBUG` | Docker property file. |
| `app.admin.username` | `admin` fallback in `@Value` | Runtime; test explicitly `admin` | `SecurityConfig.java` and test properties. |
| `app.admin.password` | Required; no production default | Test: `test-admin-password` | `SecurityConfig.java` reads it with `@Value`; test property supplies a context-only value. Compose supplies `APP_ADMIN_PASSWORD`, but no direct property placeholder is declared in the checked-in Spring files. |

## Startup Parameters & Resource Requirements

| Service | JVM/Runtime Options | Memory | Instance Count |
|---|---|---|---|
| `photoalbum-java-app` | `JAVA_OPTS="-Xmx512m -Xms256m"`; entrypoint runs `java $JAVA_OPTS -jar app.jar`; `SPRING_PROFILES_ACTIVE=docker` | JVM heap 256 MiB initial / 512 MiB maximum; no container memory limit specified | One Compose container; `restart: on-failure`; no autoscaling setting. |
| `oracle-db` | Oracle image entrypoint; `ORACLE_PASSWORD`, `APP_USER`, and `APP_USER_PASSWORD` environment variables | No Compose memory/CPU limit; README recommends at least 4 GB Docker memory for Oracle | One Compose container; healthcheck starts after 180 seconds and retries 15 times. |
| Azure AKS provisioned by `azure-setup.ps1` | No application JVM options or workload manifest specified | Node size `Standard_D8ds_v5`; no pod requests/limits found | Two AKS nodes; no application replica count or autoscaler configured. |

The Dockerfile uses Java 8 runtime images. No `-Dspring.profiles.active` or other JVM system properties are defined outside the Compose environment variable.

## Startup Dependency Chain

1. `oracle-db` starts with the Oracle Free image, mounts `oracle_data`, and runs its initialization scripts. The container healthcheck executes `healthcheck.sh` every 30 seconds with a 10-second timeout, 15 retries, and a 180-second start period.
2. `photoalbum-java-app` waits for `oracle-db` via Compose `depends_on.condition: service_healthy`, then starts with the `docker` Spring profile and the Oracle datasource URL.
3. No separate config server, discovery service, cache, message broker, readiness probe, or startup wait-for-TCP utility was found.

## Secrets & Sensitive Configuration

| Secret Reference | Type | Storage (masked) |
|---|---|---|
| `ORACLE_PASSWORD` | Oracle SYS/SYSTEM administrator password | `.env` at runtime; value `[MASKED]`; required by Compose and Oracle init scripts. |
| `APP_USER_PASSWORD` | Application Oracle schema password | `.env` at runtime; value `[MASKED]`; mapped to `SPRING_DATASOURCE_PASSWORD` for the application. |
| `APP_ADMIN_PASSWORD` | In-memory HTTP Basic admin password | `.env` at runtime; value `[MASKED]`; supplied to the Compose application container, but the checked-in Spring property files do not show a direct `APP_ADMIN_PASSWORD` placeholder mapping. |
| `spring.datasource.password` / `SPRING_DATASOURCE_PASSWORD` | Datasource credential | Environment placeholder; value `[MASKED]`; required with no property-file default. |
| `POSTGRES_ADMIN_PASSWORD` | Azure PostgreSQL administrator password | Environment variable or generated at provisioning time; written to git-ignored `.env` as `[MASKED]` in this inventory. |
| `POSTGRES_APP_PASSWORD` | Azure PostgreSQL application-user password | Environment variable or cryptographically generated by `azure-setup.ps1`; written to git-ignored `.env` as `[MASKED]`. |
| Test `app.admin.password` | Test-only credential | `application-test.properties`; value is a non-production test credential and should not be reused. |

### Secrets Provisioning Workflow

For local Compose, the operator copies `.env.example` to `.env` and replaces placeholders with strong unique values. Docker Compose interpolates the database administrator/schema credentials and application admin password; the application receives the schema password as `SPRING_DATASOURCE_PASSWORD`, while the Oracle container receives the database values directly. The Oracle startup scripts use those container variables to initialize or grant the schema user. No managed identity, RBAC binding, Key Vault access policy, vault path, or CI secret retrieval workflow is configured in the repository.

The separate `azure-setup.ps1` path generates or reads PostgreSQL administrator and application passwords, creates Azure Database for PostgreSQL Flexible Server, stores connection values in a git-ignored `.env`, and provisions AKS/ACR. That script does not create Kubernetes Secret objects or connect the current Oracle-based Spring datasource to PostgreSQL, so its generated secrets are not automatically consumed by the application.

## Feature Flags

| Flag Name | Default | Controlled By |
|---|---|---|
| None found | — | No feature-flag framework, `@ConditionalOnProperty`, `@ConditionalOnExpression`, LaunchDarkly, Unleash, or equivalent custom toggle was found. |

The `docker` and `test` Spring profiles are environment/configuration modes, not feature flags.

## Framework & Runtime Versions

| Component | Version | Source |
|---|---|---|
| Spring Boot | `2.7.18` | Parent version in `pom.xml`; README also documents this version. |
| Java language/runtime target | `8` / `1.8` | `java.version`, Maven compiler source/target, Docker images, and README. |
| Spring Framework and managed dependencies | Managed by Spring Boot `2.7.18` | `spring-boot-starter-*` dependencies in `pom.xml`. |
| Hibernate | Managed transitively by Spring Boot `2.7.18` | `spring-boot-starter-data-jpa`; Oracle dialect property. |
| Spring Security | Managed transitively by Spring Boot `2.7.18` | `spring-boot-starter-security`; legacy `authorizeRequests`/`antMatchers` DSL in `SecurityConfig.java`. |
| Thymeleaf | Managed transitively by Spring Boot | `spring-boot-starter-thymeleaf`. |
| Oracle JDBC | `ojdbc8` artifact; exact managed version inherited from Spring Boot dependency management | `pom.xml`, runtime scope. |
| Commons IO | `2.11.0` | Explicit version in `pom.xml`. |
| H2 | Managed transitively by Spring Boot | Test scope dependency in `pom.xml`. |
| Maven | `3.9.6` in the Docker build image | `Dockerfile`; host Maven version is not pinned in the repository. |
| Build base image | `maven:3.9.6-eclipse-temurin-8` | `Dockerfile`. |
| Runtime base image | `eclipse-temurin:8-jre` | `Dockerfile`. |
| Oracle container | `gvenzl/oracle-free:latest` | `docker-compose.yml`; tag is floating. |
| Azure PostgreSQL server | PostgreSQL `15` | `azure-setup.ps1`; separate Azure provisioning path, not current local Oracle Compose path. |
| AKS node image/version | Not specified | `azure-setup.ps1` selects node VM size only; no Kubernetes workload manifests are present. |

> Note: Some runtime details are inferred from scripts and documentation because no Kubernetes manifests, CI deployment workflow for the application, external secret-store configuration, or pinned transitive dependency report was found.
