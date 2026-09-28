# Upgrade Plan: Photo Album (20260928072441)

- **Generated**: 2026-09-28 09:24:41
- **HEAD Branch**: main
- **HEAD Commit ID**: 677fb65daaf550c9ef87a97ede1f9af3a5119e71

## Available Tools

**JDKs**
- JDK 8: not available (baseline will be skipped)
- JDK 25: C:\Program Files\Microsoft\jdk-25.0.4.101-hotspot\bin (required by steps 3-6)

**Build Tools**
- Maven: not found in PATH → **<TO_BE_INSTALLED>** to 3.9.6+
- Maven Wrapper: not present

## Guidelines

> Note: This upgrade requires migrating from Spring Boot 2.7.18 + Java 8 to Spring Boot 4.0+ + Java 25. Key changes include javax→jakarta migration, Spring Security DSL refactoring, and Dockerfile update.

## Options

- Working branch: appmod/java-upgrade-20260928072441
- Run tests before and after the upgrade: true

## Upgrade Goals

- Java: 8 → 25
- Spring Boot: 2.7.18 → 4.0.x
- Spring Framework: 5.3.x → 7.x
- Package namespaces: javax.* → jakarta.*

## Technology Stack

| Technology/Dependency | Current | Min Compatible | Why Incompatible |
|----------------------|---------|-----------------|-----------------|
| Java | 8 | 25 | User requested; Spring Boot 4.0 requires Java 17+ |
| Spring Boot | 2.7.18 | 4.0.0 | User requested |
| Spring Framework ⚠️ (transitive) | 5.3.x | 7.x | Driven by Spring Boot 4.0 upgrade |
| Spring Security (transitive) | 5.7.x | 7.x | Driven by Spring Boot 4.0 upgrade |
| Hibernate (transitive) ⚠️ | 5.6.x | 6.4.x | Spring Boot 2.7→3.x transition requires Hibernate 6.x |
| javax.persistence ⚠️ EOL | (legacy) | N/A | Removed in Spring Boot 3.x; replaced by jakarta.persistence |
| javax.validation ⚠️ EOL | (legacy) | N/A | Removed in Spring Boot 3.x; replaced by jakarta.validation |
| Maven | not found | 3.9.6+ | Required for build; 3.8.x is EOL; 3.9+ recommended for Java 17+ support |
| maven-compiler-plugin | 3.x (managed) | 3.11.0+ | Recommended for better Java 25 compatibility |
| commons-io | 2.11.0 | 2.15.1+ | No critical incompatibility but 2.15+ supports newer Java |
| Oracle JDBC (ojdbc8) | 8.x | 23.x+ | ojdbc8 targets JDK 8-11; ojdbc11 targets 11-17; ojdbc23 targets 17+ |

## Derived Upgrades

Based on target versions and compatibility rules:

1. **Spring Boot 2.7.18 → 3.3.0** (intermediate before 4.0)
   - Required because Spring Boot 3.x performs the javax→jakarta namespace migration
   - Also bridges to Hibernate 6.x and Spring Framework 6.x
   - Must occur before Spring Boot 4.0 upgrade

2. **Spring Boot 3.3.0 → 4.0.3+** (final target)
   - Supports Java 17+ natively; requires Java 17+ minimum
   - Brings Spring Framework 7.x, Spring Security 7.x
   - Removes deprecated APIs from 3.x series

3. **Java 8 → Java 21** (intermediate)
   - Bridge to Java 25; allows compile/test with Java 21 to verify compatibility before Java 25
   - Spring Boot 4.0 supports Java 21+

4. **Java 21 → Java 25** (final target)
   - User requested; all deps compatible at this point

5. **Maven → 3.9.6+**
   - Required for build; 3.8.x is EOL
   - Improved Java 17+ and Java 25 support

6. **Oracle JDBC ojdbc8 → ojdbc23**
   - ojdbc8 targets Java 8-11; ojdbc23 targets Java 17+
   - Required for compatibility with Java 25

7. **Spring Security DSL Migration**
   - Spring Security 6.x+ removed `authorizeRequests()`, `antMatchers()`
   - Must use `authorizeHttpRequests()`, `requestMatchers()` instead
   - Affects `SecurityConfig.java`

8. **Dockerfile**
   - Update base image from `maven:3.9.6-eclipse-temurin-8` to `maven:3.9.6-eclipse-temurin-25`
   - Update runtime from `eclipse-temurin:8-jre` to `eclipse-temurin:25-jre`

## Impact Analysis

### Dependency Changes

| File | Dependency | Current | Action | Target | Reason |
|------|-----------|---------|--------|--------|--------|
| pom.xml | spring-boot-starter-parent | 2.7.18 | upgrade | 3.3.0 | Intermediate step: enables javax→jakarta migration |
| pom.xml | java.version property | 1.8 | upgrade | 21 | Intermediate: Java 21 before final Java 25 |
| pom.xml | maven.compiler.source | 8 | upgrade | 21 | Supports Java 21 source |
| pom.xml | maven.compiler.target | 8 | upgrade | 21 | Supports Java 21 bytecode |
| pom.xml | spring-boot-starter-parent | 3.3.0 | upgrade | 4.0.3 | User requested final target |
| pom.xml | java.version property | 21 | upgrade | 25 | User requested final target |
| pom.xml | maven.compiler.source | 21 | upgrade | 25 | Supports Java 25 source |
| pom.xml | maven.compiler.target | 21 | upgrade | 25 | Supports Java 25 bytecode |
| pom.xml | com.oracle.database.jdbc:ojdbc8 | (managed) | replace | ojdbc11 (SB 3.x) then ojdbc23 (SB 4.x) | ojdbc8 targets Java 8-11; ojdbc23 targets Java 17+ |

### Source Code Changes

| File | Location | Current | Required Change | Reason |
|------|----------|---------|----------------|--------|
| src/main/java/com/photoalbum/model/Photo.java | import lines 3-6 | javax.persistence.*<br/>javax.validation.* | Replace with:<br/>jakarta.persistence.*<br/>jakarta.validation.* | Jakarta EE 9+ namespace |
| src/main/java/com/photoalbum/config/SecurityConfig.java | imports & method | org.springframework.security.config.annotation.web.builders.HttpSecurity<br/>.csrf().disable()<br/>.sessionManagement()...sessionCreationPolicy(...)<br/>.and()<br/>.authorizeRequests()<br/>.antMatchers(...)<br/>.and()<br/>.httpBasic() | Rewrite to Spring Security 6+ DSL:<br/>.csrf(c -> c.disable())<br/>.sessionManagement(s -> s.sessionCreationPolicy(...))<br/>.authorizeHttpRequests(a -> a<br/>.requestMatchers(HttpMethod.POST, ...)<br/>.authenticated()<br/>.anyRequest().permitAll())<br/>.httpBasic(Customizer.withDefaults()) | Spring Security 6.0+ removed deprecated builders; lambda DSL is preferred |

### Configuration Changes

No application.properties/application.yml changes required for this project.

### CI/CD Changes

| File | Location | Current | Required Change |
|------|----------|---------|----------------|
| Dockerfile | line 2 | FROM maven:3.9.6-eclipse-temurin-8 AS build | Change to: maven:3.9.6-eclipse-temurin-25 |
| Dockerfile | line 17 | FROM eclipse-temurin:8-jre | Change to: eclipse-temurin:25-jre |

### Risks & Warnings

- **SecurityConfig DSL Refactoring**: The Spring Security 6.0+ DSL is a non-trivial rewrite. The old `authorizeRequests()` + `antMatchers()` + `.and()` chaining pattern must be replaced with lambda DSL. This is a breaking change with no backward compatibility. **Mitigation**: The SecurityConfig in this project is relatively simple (2 security rules). Test with the existing integration tests if available; add a security smoke test if none exist to verify HTTP Basic auth still works.

- **Oracle JDBC Driver Version Jump**: Migrating from ojdbc8 (Java 8-11) to ojdbc23 (Java 17+) is a major version jump. While functional compatibility is generally high, some internal API changes may occur. **Mitigation**: Verify Oracle JDBC connectivity in post-upgrade testing. Add a test that connects to Oracle DB if infrastructure is available.

- **Intermediate Java 21**: Using Java 21 as an intermediate before Java 25 is to ensure compatibility; Java 25 is very new and using an LTS intermediate reduces risk.

- **Maven Installation**: Maven must be installed. The system does not have Maven in PATH. **Mitigation**: Install Maven 3.9.6+ using the upgrade tool in Step 1.

## Upgrade Steps

- **Step 1: Install Maven 3.9.6+**
  - **Rationale**: Maven is not available on the system; it is required for the build to proceed.
  - **Changes to Make**: Install Maven 3.9.6+ (system default, not project-specific)
  - **Verification**: `mvn --version` should output Maven 3.9.6+

- **Step 2: Setup Baseline (SKIPPED)**
  - **Rationale**: Base JDK (Java 8) is not available on the system. Baseline compilation and test run will be skipped.
  - **Changes to Make**: None
  - **Verification**: N/A (skipped due to missing base JDK)

- **Step 3: Upgrade to Spring Boot 3.3.0 and Java 21 (Intermediate)**
  - **Rationale**: Spring Boot 3.3.0 is the first version in the 3.x line that performs javax→jakarta namespace migration. Java 21 is the first intermediate LTS that supports Spring Boot 3.x. This step bridges to the final targets.
  - **Changes to Make**: 
    - Update spring-boot-starter-parent from 2.7.18 to 3.3.0
    - Update java.version, maven.compiler.source, maven.compiler.target to 21
  - **Verification**: `mvn clean test-compile -q` with Java 21 JDK. Expected: compilation SUCCESS (main + test code). Tests may fail due to API changes; fix in subsequent steps.

- **Step 4: Migrate javax.* to jakarta.* Packages**
  - **Rationale**: Spring Boot 3.x uses Jakarta EE 9+ which requires javax→jakarta namespace migration. This migration is automatic in BOM transitive deps but must be applied to source code manually.
  - **Changes to Make**: 
    - Update imports in Photo.java: javax.persistence.* → jakarta.persistence.*; javax.validation.* → jakarta.validation.*
  - **Verification**: `mvn clean test-compile -q` with Java 21. Expected: compilation SUCCESS.

- **Step 5: Update Spring Security Configuration to 6.x DSL**
  - **Rationale**: Spring Security 6.0+ (brought in by Spring Boot 3.x) removed deprecated builders (authorizeRequests, antMatchers). Must use lambda DSL with authorizeHttpRequests, requestMatchers.
  - **Changes to Make**: 
    - Refactor SecurityConfig.java to use new Spring Security 6.x DSL (see Impact Analysis)
  - **Verification**: `mvn clean test-compile -q` with Java 21. Expected: compilation SUCCESS.

- **Step 6: Upgrade Oracle JDBC Driver to ojdbc11 (for SB 3.x)**
  - **Rationale**: ojdbc8 targets Java 8-11. Spring Boot 3.3 (managing dependencies) will request ojdbc11 or later. Explicitly manage this dependency for clarity.
  - **Changes to Make**: 
    - No explicit action needed; Spring Boot 3.3 BOM manages ojdbc version. Verify pom.xml dependency:list shows ojdbc11 or later.
  - **Verification**: `mvn dependency:list -DexcludeTransitive=true | grep ojdbc` should show ojdbc11 or later. `mvn clean test-compile -q` should succeed.

- **Step 7: Upgrade to Spring Boot 4.0.3 (Final)**
  - **Rationale**: User requested Spring Boot 4.0+. Spring Boot 4.0 requires Java 17+ (we'll use 21 as intermediate, then 25 as final).
  - **Changes to Make**: 
    - Update spring-boot-starter-parent from 3.3.0 to 4.0.3
  - **Verification**: `mvn clean test-compile -q` with Java 21. Expected: compilation SUCCESS.

- **Step 8: Upgrade to Java 25 (Final Target)**
  - **Rationale**: User requested Java 25 as the final target.
  - **Changes to Make**: 
    - Update java.version, maven.compiler.source, maven.compiler.target to 25
    - Update Dockerfile: maven:3.9.6-eclipse-temurin-8 → maven:3.9.6-eclipse-temurin-25 and eclipse-temurin:8-jre → eclipse-temurin:25-jre
  - **Verification**: `mvn clean test-compile -q` with Java 25. Expected: compilation SUCCESS.

- **Step 9: Update Oracle JDBC Driver to ojdbc23 (for Java 25)**
  - **Rationale**: ojdbc8 and ojdbc11 target Java 8-17. For Java 25, use ojdbc23+ which targets Java 17+. Spring Boot 4.0 may manage this automatically but we verify.
  - **Changes to Make**: 
    - Verify pom.xml dependency:list shows ojdbc23 or later.
    - If not, explicitly upgrade Oracle JDBC dependency version in pom.xml.
  - **Verification**: `mvn dependency:list -DexcludeTransitive=true | grep ojdbc` should show ojdbc23 or later. `mvn clean test-compile -q` should succeed.

- **Step 10: Final Validation**
  - **Rationale**: Verify all upgrade goals are met, all tests pass, and the project is fully compatible with Java 25 + Spring Boot 4.0.
  - **Changes to Make**: Fix any remaining test failures. Resolve all TODOs and workarounds. Ensure 100% test pass rate.
  - **Verification**: `mvn clean test -q` with Java 25. Expected: 100% test pass rate or ≥ baseline (baseline unknown, so 100% required).

---

**Summary**: This plan breaks the upgrade into 10 steps that incrementally migrate the project from Java 8 + Spring Boot 2.7.18 to Java 25 + Spring Boot 4.0.3. Key milestones are: (1) Maven installation, (2) Spring Boot 3.x + Java 21 intermediate, (3) javax→jakarta migration, (4) Spring Security DSL update, (5) Spring Boot 4.0 + Java 25 final. Each step is designed to leave the project in a compilable state.
