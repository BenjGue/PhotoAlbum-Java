---
name: java-upgrade
description: >-
  Transformation rules and verification steps for upgrading PhotoAlbum-Java
  from Java 8 + Spring Boot 2.7.18 (Oracle, Thymeleaf, Spring Security) to
  Java 25 + Spring Boot 4.0, ready for Azure Container Apps.
  WHEN: upgrade Java 8 to 17/21/25, migrate Spring Boot 2.7 to 3.x or 4.0,
  javax to jakarta, Spring Security 5 to 6/7 DSL migration, antMatchers or
  authorizeRequests removal, fix CVE in Java dependencies, update Dockerfile
  to Temurin 25, Hibernate 6/7 upgrade issues.
  NOT for: Oracle to PostgreSQL data migration, greenfield Java services,
  .NET projects, Azure infrastructure provisioning (Bicep/azd).
---

# PhotoAlbum-Java — Java 8 / Boot 2.7 → Java 25 / Boot 4.0 Upgrade Rules

## Purpose

Packages the version-step strategy, API mappings, and verification checklist for a
behavior-preserving upgrade of PhotoAlbum-Java to Java 25 + Spring Boot 4.0.

## Prerequisites

- `ASSESSMENT.md` exists and is approved.
- JDK 17 **and** JDK 25 available, Maven ≥ 3.9 — verify: `java -version`, `mvn -version`.
- Docker available for the Oracle Free container (`docker compose up`) used by the smoke test.

## Procedure

### Step 1 — Upgrade incrementally (never skip)

| Step | From → To | Key risk at this step |
|---|---|---|
| 1 | Java 8 → 17 (Boot 2.7.18) | Stale `maven.compiler.source/target=8` in `pom.xml`; JAXB-style removed JEE modules |
| 2 | Boot 2.7 → 3.5 (Java 17) | `javax → jakarta`, Spring Security 6, Hibernate 6 |
| 3 | Java 17 → 25 (Boot 3.5) | Bytecode libs / agents (ByteBuddy, Mockito) must support Java 25 |
| 4 | Boot 3.5 → 4.0 (Java 25) | Spring Security 7 (lambda DSL only), Hibernate 7, Jackson 3, starter renames |
| 5 | `Dockerfile` | `maven:3.9-eclipse-temurin-25` build + `eclipse-temurin:25-jre` runtime |

```powershell
mvn -q clean verify   # run after EVERY step; fix before proceeding
```

### Step 2 — Apply transformation rules

| Source pattern | Target pattern | Notes |
|---|---|---|
| `<java.version>1.8</java.version>` + `maven.compiler.source/target=8` | `<java.version>25</java.version>`, delete the `maven.compiler.*` properties | The Boot parent derives `maven.compiler.release` from `java.version` |
| `javax.persistence.*` / `javax.validation.*` (`model/Photo.java`) | `jakarta.persistence.*` / `jakarta.validation.*` | Leave `javax.imageio.*` (JDK) untouched |
| `.csrf().disable()`, `.sessionManagement()...and()`, `.httpBasic()` | `.csrf(c -> c.disable())`, `.sessionManagement(s -> s.sessionCreationPolicy(STATELESS))`, `.httpBasic(Customizer.withDefaults())` | `and()` and the non-lambda DSL are removed in Spring Security 7 |
| `.authorizeRequests().antMatchers(HttpMethod.POST, "/upload", "/detail/*/delete")` | `.authorizeHttpRequests(a -> a.requestMatchers(HttpMethod.POST, "/upload", "/detail/*/delete").authenticated().anyRequest().permitAll())` | Keep the exact same public/protected paths |
| `spring.jpa.database-platform=org.hibernate.dialect.OracleDialect` | Remove the property (dialect auto-detected) | Explicit dialects log deprecation warnings on Hibernate 6+ |
| `com.oracle.database.jdbc:ojdbc8` | `ojdbc11` (or `ojdbc17`), version managed by the Boot BOM | `ojdbc8` targets Java 8 |
| `commons-io:commons-io:2.11.0` | Latest `2.x` | 2.11.0 is affected by CVE-2024-47554 (fixed in 2.14.0) |
| `spring-boot-starter-web` (Boot 4.0) | `spring-boot-starter-webmvc` | Old name kept as deprecated alias — check the Boot 4.0 migration guide |
| `spring.jpa.hibernate.ddl-auto=create` | `validate` (or `update` for the workshop) via env var `SPRING_JPA_HIBERNATE_DDL_AUTO` | `create` wipes all photos on every container restart |
| `logging.level.org.springframework.web=DEBUG`, `spring.jpa.show-sql=true` | `INFO` / `false` in the cloud profile | Log noise and cost in Log Analytics |

### Step 3 — Verify

- [ ] `mvn clean verify` passes on JDK 25 (H2, profile `test`)
- [ ] `grep -r "javax\." src/` returns only `javax.imageio` (JDK)
- [ ] No `.and()`, `antMatchers`, or `authorizeRequests` left in `SecurityConfig.java`
- [ ] `docker build .` succeeds with the Temurin 25 images
- [ ] App starts against Oracle Free; `GET /` returns HTTP 200; anonymous `POST /upload` is rejected (401)
- [ ] CVE assessment clean of criticals (`appmod-java-cve-assessment`)

## Common Pitfalls

- ⚠️ **Oracle-only native SQL** in `PhotoRepository` (`ROWNUM`, `TO_CHAR`, `NVL`, `RANK() OVER`) keeps working on Oracle but blocks the PostgreSQL migration of Challenge 3 — prefer JPQL + `Pageable`/`Sort` when that migration happens.
- ⚠️ **Oracle column definitions** in `Photo.java` (`NUMBER(19,0)`, `TIMESTAMP DEFAULT SYSTIMESTAMP`) are not portable — drop `columnDefinition` before switching engines.
- ⚠️ **`@Lob byte[] photoData`** maps to a PostgreSQL large object (`oid`), not `bytea`, with Hibernate — use `@JdbcTypeCode(SqlTypes.VARBINARY)` instead of `@Lob` when targeting PostgreSQL.
- ⚠️ **Hibernate 6+ strictness** on native queries returning extra columns (`RN`, `SIZE_RANK`) mapped to `Photo` — run the navigation and pagination paths, not only `contextLoads()`.
- ⚠️ **Jackson 3 in Boot 4.0** — `com.fasterxml.jackson.databind` becomes `tools.jackson.databind`; check any custom `ObjectMapper` (annotations keep their package).
- ⚠️ **Required secret** — `app.admin.password` has no default, so the app fails to start if the env var is missing in Container Apps.

## References

- Spring Boot 3.0 and 4.0 migration guides; Spring Security 6 and 7 migration guides
- OpenRewrite recipes: `org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_5`, `org.openrewrite.java.migrate.UpgradeToJava25`
- appmod assessment report (linked from `ASSESSMENT.md`)
