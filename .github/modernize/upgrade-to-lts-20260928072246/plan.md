# Java 25 and Spring Boot 4.x Upgrade Plan

## Overview

This upgrade plan guides the migration of your Java application to the latest LTS versions:
- **Java**: 25
- **Spring Boot**: 4.x
- **Spring Framework**: 7.x
- **Jakarta EE**: Latest (with javax.* → jakarta.* migration)

These versions represent the latest stable releases with long-term support.

## Key Changes

### Java 25
- Latest LTS release with improved performance and security features
- New language features and JVM optimizations

### Spring Boot 4.x
- Requires Java 25 minimum
- Includes Spring Framework 7.x
- Full Jakarta EE support (javax.* → jakarta.*)
- Enhanced observability, native compilation, and cloud support

### Spring Framework 7.x
- Modern Java features utilization
- Improved modularity and performance
- Complete Jakarta EE namespace support

## Tasks

See `.metadata/tasks.json` for the detailed task breakdown.

### 001-upgrade-java-spring-boot
Upgrade JDK to Java 25, Spring Boot to 4.x, Spring Framework to 7.x, and migrate javax.* to jakarta.* packages.

**Success Criteria**:
- Project builds successfully
- All unit tests pass

## Migration Path

1. Update JDK to Java 25
2. Update Spring Boot to 4.x (automatically brings Spring Framework 7.x)
3. Migrate javax.* imports to jakarta.* namespace
4. Update all dependencies to compatible versions
5. Run tests and validate the build

## References

- [Java 25 Release Notes](https://openjdk.java.net)
- [Spring Boot 4.0 Release Notes](https://spring.io/projects/spring-boot)
- [Spring Framework 7.x Documentation](https://spring.io/projects/spring-framework)
- [Jakarta EE Migration Guide](https://jakarta.ee)
