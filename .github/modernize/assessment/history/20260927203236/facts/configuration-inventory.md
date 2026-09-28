# Configuration & Externalized Settings Inventory

The application uses Maven, Spring properties, Docker Compose, a `.env` template, and Oracle initialization scripts as its configuration sources. Local runtime secrets are externalized through environment variables; no config server or managed secret-store integration is configured in the application.

## Configuration Sources

| Source | Type | Path/Location | Notes |
|---|---|---|---|
| Maven project descriptor | Build configuration | `pom.xml` | Spring Boot parent, Java compiler target, dependencies, and packaging plugin. |
| Default Spring configuration | Application properties | `src/main/resources/application.properties` | Port, encoding, Oracle datasource, JPA, upload limits, and logging. |
| Docker Spring configuration | Runtime profile properties | `src/main/resources/application-docker.properties` | Docker datasource and logging overrides; profile is selected by Compose. |
| Test Spring configuration | Test properties | `src/test/resources/application-test.properties` | H2 in-memory datasource, test JPA settings, test upload path, and test admin credentials. |
| Compose environment | Container orchestration | `docker-compose.yml` | Oracle and application services, environment variables, ports, volume, network, healthcheck, and dependency condition. |
| Environment template | Secret/configuration template | `.env.example` | Required local Oracle and application-admin values. Real `.env` is expected to be git-ignored. |
| Container build/runtime | Dockerfile | `Dockerfile` | Maven/Eclipse Temurin 8 build image, Java 8 runtime image, and `JAVA_OPTS`. |
| Oracle initialization | Database startup scripts | `oracle-init/*` | User creation, grants, verification, and healthcheck scripts; credentials come from container environment. |
| Azure provisioning | Infrastructure script | `azure-setup.ps1` | Creates ACR, AKS, and PostgreSQL Flexible Server and writes Azure values to `.env`; it is not application runtime configuration. |
| Azure cleanup | Infrastructure lifecycle script | `azure-reset.ps1` | Removes ACR images and an AKS namespace. |
| External config server | Not found | — | No Spring Cloud Config, Consul, App Configuration, or external Git configuration URI. |
| Secret store | Not found | — | No Key Vault, HashiCorp Vault, AWS Secrets Manager, encrypted properties, or sealed secrets. |
| Kubernetes configuration | Not found | — | No Deployment, ConfigMap, Secret, Helm chart, or readiness manifest. |

## Build Profiles

| Profile | Activation | Purpose | Key Dependencies/Plugins |
|---|---|---|---|
| Maven default | Automatic when Maven runs without `-P` | Builds the single `photo-album` JAR | Spring Boot parent `2.7.18`, Java source/target 8, `spring-boot-maven-plugin`. |
| Maven named profiles | None declared | No project-specific dev, test, Docker, or cloud Maven profiles | No conditional dependencies or plugins. |
| Docker build stage | Selected by `docker build` or Compose | Resolves dependencies and packages the application | `maven:3.9.6-eclipse-temurin-8`; `mvn clean package -DskipTests`. |

## Runtime Profiles

| Profile | Activation Method | Config Files | Key Overrides |
|---|---|---|---|
| Default | Spring default when no profile is active | `application.properties` | Oracle URL default, port 8080, DEBUG application and web logging. |
| `docker` | `SPRING_PROFILES_ACTIVE=docker` in `docker-compose.yml` | `application.properties` plus `application-docker.properties` | Docker datasource bindings, port 8080, application INFO, web WARN, Hibernate SQL DEBUG. |
| `test` | Test classpath/property selection; no global active profile found | `src/test/resources/application-test.properties` | H2 in-memory database, `create-drop`, test upload path, and test-only admin credentials. |

No `@Profile` annotations or combined profile activation were found. The Docker profile is explicitly selected by Compose; the test file is test configuration rather than a production profile.

## Properties Inventory

### Application and server

| Property Key | Default | Profiles | Source |
|---|---|---|---|
| `server.port` | `8080` | Default, docker | Spring property files. |
| `server.servlet.encoding.charset` | `UTF-8` | Default, docker | Spring property files. |
| `server.servlet.encoding.enabled` | `true` | Default, docker | Spring property files. |
| `server.servlet.encoding.force` | `true` | Default, docker | Spring property files. |

### Database and JPA

| Property Key | Default | Profiles | Source |
|---|---|---|---|
| `spring.datasource.url` | `${SPRING_DATASOURCE_URL:jdbc:oracle:thin:@oracle-db:1521/FREEPDB1}` | Default, docker | `application.properties`, `application-docker.properties`; Compose supplies the environment value. |
| `spring.datasource.username` | `${SPRING_DATASOURCE_USERNAME:photoalbum}` | Default, docker | Spring properties; Compose maps `APP_USER`. |
| `spring.datasource.password` | Required; no default | Default, docker | `${SPRING_DATASOURCE_PASSWORD}`; Compose maps `APP_USER_PASSWORD`. |
| `spring.datasource.driver-class-name` | `oracle.jdbc.OracleDriver` | Default, docker | Spring property files. |
| `spring.jpa.database-platform` | `org.hibernate.dialect.OracleDialect` | Default, docker; H2 dialect in test | Spring property files. |
| `spring.jpa.hibernate.ddl-auto` | `create` | Default, docker; `create-drop` in test | Spring property files. |
| `spring.jpa.show-sql` | `true` | Default, docker; `false` in test | Spring property files. |
| `spring.jpa.properties.hibernate.format_sql` | `true` | Default, docker | Spring property files. |
| `spring.datasource.url` | `jdbc:h2:mem:testdb` | Test | `application-test.properties`. |
| `spring.datasource.driver-class-name` | `org.h2.Driver` | Test | `application-test.properties`. |
| `spring.datasource.username` | `sa` | Test | `application-test.properties`. |
| `spring.datasource.password` | Empty | Test | `application-test.properties`. |

### Upload and application settings

| Property Key | Default | Profiles | Source |
|---|---|---|---|
| `spring.servlet.multipart.max-file-size` | `10MB` | Default, docker | Spring property files. |
| `spring.servlet.multipart.max-request-size` | `50MB` | Default, docker | Spring property files. |
| `app.file-upload.max-file-size-bytes` | `10485760` (10 MiB) | Default, docker, test | Spring property files. |
| `app.file-upload.allowed-mime-types` | `image/jpeg,image/png,image/gif,image/webp` | Default, docker, test | Spring property files. |
| `app.file-upload.max-files-per-upload` | `10` | Default, docker, test | Spring property files. |
| `app.file-upload.upload-path` | Not set in production | Test: `target/test-uploads` | Test property only. |

### Logging and security

| Property Key | Default | Profiles | Source |
|---|---|---|---|
| `logging.level.com.photoalbum` | `DEBUG` | Default; `INFO` in docker | Spring property files. |
| `logging.level.org.springframework.web` | `DEBUG` | Default; `WARN` in docker | Spring property files. |
| `logging.level.org.hibernate.SQL` | Not set | Docker: `DEBUG` | Docker property file. |
| `app.admin.username` | `admin` fallback | Runtime; test explicitly `admin` | `SecurityConfig.java` `@Value`; Compose supplies `APP_ADMIN_USERNAME` only as an environment variable and no direct placeholder is declared in checked-in properties. |
| `app.admin.password` | Required; no production default | Test: `test-admin-password` | `SecurityConfig.java` `@Value`; Compose supplies `APP_ADMIN_PASSWORD` without a checked-in property placeholder. |

## Startup Parameters & Resource Requirements

| Service | JVM/Runtime Options | Memory | Instance Count |
|---|---|---|---|
| `photoalbum-java-app` | `JAVA_OPTS="-Xmx512m -Xms256m"`; entrypoint runs `java $JAVA_OPTS -jar app.jar`; `SPRING_PROFILES_ACTIVE=docker` | JVM heap 256 MiB initial / 512 MiB maximum; no container limit | One Compose container; `restart: on-failure`; no autoscaling. |
| `oracle-db` | Oracle image entrypoint; `ORACLE_PASSWORD`, `APP_USER`, `APP_USER_PASSWORD` | No Compose limit; README recommends at least 4 GB Docker memory | One Compose container; healthcheck has 180-second start period and 15 retries. |
| Azure AKS path | No application JVM options or workload manifest | Node size `Standard_D8ds_v5`; no pod requests/limits | Two AKS nodes; no replica or autoscaler configuration. |

No JVM `-D` profile parameters, Kubernetes resource settings, or application replica settings were found.

## Startup Dependency Chain

1. `oracle-db` starts with `gvenzl/oracle-free:latest`, mounts `oracle_data`, and runs initialization scripts. `healthcheck.sh` runs every 30 seconds with a 10-second timeout, 15 retries, and a 180-second start period.
2. `photoalbum-java-app` waits for `oracle-db` through Compose `depends_on.condition: service_healthy`, then starts with the `docker` Spring profile and Oracle datasource settings.
3. No separate config server, discovery service, cache, message broker, wait-for-TCP utility, actuator readiness probe, or custom application healthcheck was found.

## Secrets & Sensitive Configuration

| Secret Reference | Type | Storage (masked) |
|---|---|---|
| `ORACLE_PASSWORD` | Oracle administrator password | Runtime `.env`; `[MASKED]`; required by Compose and Oracle initialization. |
| `APP_USER_PASSWORD` | Application Oracle schema password | Runtime `.env`; `[MASKED]`; mapped to `SPRING_DATASOURCE_PASSWORD`. |
| `APP_ADMIN_PASSWORD` | In-memory application admin password | Runtime `.env`; `[MASKED]`; passed to the application container. |
| `spring.datasource.password` / `SPRING_DATASOURCE_PASSWORD` | Datasource credential | Environment placeholder; `[MASKED]`; no property-file default. |
| `POSTGRES_ADMIN_PASSWORD` | Azure PostgreSQL administrator password | Environment or generated by `azure-setup.ps1`; `[MASKED]`; written to git-ignored `.env`. |
| `POSTGRES_APP_PASSWORD` | Azure PostgreSQL application-user password | Environment or generated by `azure-setup.ps1`; `[MASKED]`; written to git-ignored `.env`. |
| Test `app.admin.password` | Test-only credential | `application-test.properties`; non-production test value; `[MASKED]`. |

### Secrets Provisioning Workflow

For local Compose, the operator copies `.env.example` to `.env` and replaces placeholders with strong values. Compose interpolates Oracle administrator/schema credentials and the application admin password; the application receives the schema password as `SPRING_DATASOURCE_PASSWORD`, and Oracle receives its initialization values directly. Oracle startup scripts use those variables to create or grant the schema user.

The separate `azure-setup.ps1` workflow generates or reads PostgreSQL credentials, creates Azure PostgreSQL Flexible Server, and writes PostgreSQL connection values plus resource identifiers to the git-ignored `.env`. It does not create Kubernetes Secret objects, configure managed identity/RBAC for secrets, or map the current Oracle Spring datasource to PostgreSQL. No Key Vault or CI secret retrieval workflow is configured.

## Feature Flags

| Flag Name | Default | Controlled By |
|---|---|---|
| None found | — | No feature-flag framework, `@ConditionalOnProperty`, `@ConditionalOnExpression`, LaunchDarkly, Unleash, or equivalent custom toggle was found. |

The `docker` and `test` Spring profiles are environment modes, not feature flags.

## Framework & Runtime Versions

| Component | Version | Source |
|---|---|---|
| Spring Boot | `2.7.18` | Parent version in `pom.xml`. |
| Java language/runtime target | `8` / `1.8` | Maven properties, Docker images, and README. |
| Hibernate | Managed by Spring Boot `2.7.18` | `spring-boot-starter-data-jpa`. |
| Spring Security | Managed by Spring Boot `2.7.18` | `spring-boot-starter-security`. |
| Thymeleaf | Managed by Spring Boot | `spring-boot-starter-thymeleaf`. |
| Oracle JDBC | `ojdbc8`; exact version inherited from Boot dependency management | `pom.xml`, runtime scope. |
| Commons IO | `2.11.0` | Explicit `pom.xml` version. |
| H2 | Managed by Spring Boot | Test-scope dependency. |
| Maven | `3.9.6` in Docker build image | `Dockerfile`; host Maven is not pinned. |
| Build base image | `maven:3.9.6-eclipse-temurin-8` | `Dockerfile`. |
| Runtime base image | `eclipse-temurin:8-jre` | `Dockerfile`. |
| Oracle container | `gvenzl/oracle-free:latest` | `docker-compose.yml`; floating tag. |
| Azure PostgreSQL | `15` | `azure-setup.ps1`; separate provisioning path. |
| AKS node VM size | `Standard_D8ds_v5` | `azure-setup.ps1`; workload image/version not specified. |

> Note: Some runtime details are inferred from scripts and documentation because no Kubernetes manifests, application deployment workflow, pinned transitive dependency report, or external secret-store configuration was found.
