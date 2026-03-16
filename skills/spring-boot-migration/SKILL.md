---
name: spring-boot-migration
description: >-
  Spring Boot version migration guide covering 2.7 to 4.0.
  Use when upgrading Spring Boot versions, migrating Security configuration,
  adopting virtual threads, structured logging, or reviewing version compatibility.
---

# Spring Boot Migration Rules

## 1. Version Overview

| Version | Release    | Spring Framework | Java Min | Key Theme                              |
| ------- | ---------- | ---------------- | -------- | -------------------------------------- |
| 2.7     | 2022-05    | 5.3              | 8        | Last 2.x release, bridge to 3.x       |
| 3.0     | 2022-11    | 6.0              | 17       | Jakarta EE 9+, AOT, Observation       |
| 3.1     | 2023-05    | 6.0              | 17       | Docker Compose support, testcontainers |
| 3.2     | 2023-11    | 6.1              | 17       | Virtual threads, RestClient, CDS       |
| 3.3     | 2024-05    | 6.1              | 17       | CDS auto-setup, improved Testcontainers|
| 3.4     | 2024-11    | 6.2              | 17       | Structured logging, fallback beans     |
| 4.0     | 2025-11    | 7.0              | 17       | Jakarta EE 11, Jackson 3, built-in resilience, API versioning |

---

## 2. Spring Boot 2.7 → 3.0

### javax → jakarta Namespace Migration

This is the **largest breaking change** in Spring Boot history.

```java
// Before (Boot 2.x)
import javax.persistence.*;
import javax.servlet.*;
import javax.validation.constraints.*;
import javax.annotation.*;

// After (Boot 3.x)
import jakarta.persistence.*;
import jakarta.servlet.*;
import jakarta.validation.constraints.*;
import jakarta.annotation.*;
```

### Automated Migration Tools

| Tool                          | Approach            | Coverage               |
| ----------------------------- | ------------------- | ---------------------- |
| IntelliJ IDEA refactor        | IDE search-replace  | Import statements      |
| OpenRewrite                   | AST-based migration | Full source + config   |
| `spring-boot-migrator` (SBM)  | Spring-specific     | Boot-aware migration   |

```bash
# OpenRewrite — recommended for large projects
# Add to build.gradle.kts
plugins {
    id("org.openrewrite.rewrite") version "latest"
}
rewrite {
    activeRecipe("org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_0")
}
# Run: ./gradlew rewriteRun
```

### Spring Security Changes

```java
// Boot 2.x: WebSecurityConfigurerAdapter (REMOVED in 3.x)
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
            .antMatchers("/api/**").authenticated();
    }
}

// Boot 3.x: Component-based SecurityFilterChain
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/**").authenticated()
            )
            .build();
    }
}
```

### Key API Changes

| Boot 2.x                              | Boot 3.x                              |
| -------------------------------------- | -------------------------------------- |
| `WebSecurityConfigurerAdapter`         | `SecurityFilterChain` bean             |
| `.antMatchers()`                       | `.requestMatchers()`                   |
| `.authorizeRequests()`                 | `.authorizeHttpRequests()`             |
| `spring.redis.*`                       | `spring.data.redis.*`                  |
| `spring.elasticsearch.*`              | `spring.elasticsearch.uris`            |
| `@ConstructorBinding` (on class)       | `@ConstructorBinding` (on constructor) |
| Hibernate 5.x                          | Hibernate 6.x                          |
| `javax.persistence.*`                  | `jakarta.persistence.*`                |

### Hibernate 6 Breaking Changes

```java
// ID generation strategy change
// Boot 2.x (Hibernate 5): IDENTITY was often the effective default
// Boot 3.x (Hibernate 6): SEQUENCE is preferred

// Explicit IDENTITY strategy for MySQL
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

// HQL/JPQL changes
// Boot 2.x: implicit type coercion in queries was more lenient
// Boot 3.x: stricter type checking — cast explicitly if needed
```

### Properties Migration

```bash
# Use properties migrator to detect renamed properties
# build.gradle.kts
runtimeOnly("org.springframework.boot:spring-boot-properties-migrator")
# Logs warnings for renamed/removed properties at startup
# REMOVE this dependency before production deployment
```

### Migration Notes

- **Upgrade Java to 17 first** — do not upgrade Boot and Java simultaneously
- Run `spring-boot-properties-migrator` to detect renamed properties
- Review all `@ConstructorBinding` usage — annotation semantics changed
- Verify all third-party starters support Boot 3.x
- Test thoroughly — Hibernate 6 query behavior may differ subtly

---

## 3. Spring Boot 3.0 → 3.1

### Docker Compose Integration

```yaml
# compose.yaml in project root — Boot auto-detects and starts containers
services:
  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
```

```yaml
# application.yml
spring:
  docker:
    compose:
      enabled: true
      lifecycle-management: start-and-stop
```

### Testcontainers Integration

```java
// @ServiceConnection auto-configures connection properties
@SpringBootTest
@Testcontainers
class IntegrationTest {
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    // No @DynamicPropertySource needed — @ServiceConnection handles it
}
```

### Migration Notes

- `@ServiceConnection` replaces `@DynamicPropertySource` for supported containers
- Docker Compose integration requires Docker to be running locally
- Review `spring.docker.compose.skip.in-tests=true` for CI environments

---

## 4. Spring Boot 3.1 → 3.2

### Virtual Threads Support

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
```

- Tomcat/Jetty automatically use virtual threads for request handling
- `@Async` methods run on virtual threads
- `@Scheduled` methods run on virtual threads
- No code changes required — configuration only
- Requires Java 21+

### RestClient Auto-Configuration

```java
// Boot 3.2+: RestClient.Builder is auto-configured
@Service
public class UserApiClient {
    private final RestClient restClient;

    public UserApiClient(RestClient.Builder builder) {
        this.restClient = builder
            .baseUrl("https://api.example.com")
            .build();
    }
}
```

### Class Data Sharing (CDS)

```bash
# CDS improves startup time by sharing class metadata
# Boot 3.2 adds buildpack support for CDS
# In production Dockerfile:
FROM eclipse-temurin:21-jre-alpine
COPY app.jar app.jar
# Extract CDS archive
RUN java -Djarmode=tools -jar app.jar extract --layers --launcher
RUN java -XX:ArchiveClassesAtExit=app.jsa -Dspring.context.exit=onRefresh -jar app.jar
ENTRYPOINT ["java", "-XX:SharedArchiveFile=app.jsa", "-jar", "app.jar"]
```

### SSL Bundle Auto-Configuration

```yaml
# Centralized SSL configuration
spring:
  ssl:
    bundle:
      jks:
        web-server:
          key:
            alias: my-app
          keystore:
            location: classpath:keystore.p12
            password: ${SSL_KEYSTORE_PASSWORD}
```

### Migration Notes

- Virtual threads require Java 21+ — set `spring.threads.virtual.enabled` only in Java 21+ environments
- Review `synchronized` blocks when enabling virtual threads (pinning issue on Java 21–23, fixed in Java 24+)
- `RestClient.Builder` replaces manual `RestClient.create()` — use auto-configured builder
- CDS requires a training run to generate the archive — add to Docker build process

---

## 5. Spring Boot 3.2 → 3.3

### Improved CDS (Auto-Setup)

```bash
# 3.3+: simplified CDS with spring-boot:run support
./gradlew bootRun --args='--spring.context.exit=onRefresh -XX:ArchiveClassesAtExit=app.jsa'
# Then run with:
java -XX:SharedArchiveFile=app.jsa -jar app.jar
```

### Enhanced Testcontainers

```java
// @ContainerConnection — simplified container-based testing
@TestConfiguration(proxyBeanMethods = false)
class TestContainersConfig {
    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgres() {
        return new PostgreSQLContainer<>("postgres:16-alpine");
    }

    @Bean
    @ServiceConnection
    GenericContainer<?> redis() {
        return new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);
    }
}
```

### Migration Notes

- Minimal breaking changes from 3.2 — mostly additive improvements
- Review CDS configuration if previously set up manually

---

## 6. Spring Boot 3.3 → 3.4

### Structured Logging

```yaml
# application.yml — enable structured logging output
logging:
  structured:
    format:
      console: ecs  # Elastic Common Schema
      # Or: logfmt, gelf
```

```json
// Output example (ECS format)
{
  "@timestamp": "2024-11-15T10:30:45.123Z",
  "log.level": "INFO",
  "message": "User login successful",
  "service.name": "user-service",
  "trace.id": "abc123"
}
```

### Fallback Beans (From Framework 6.2)

```java
@Configuration
public class CacheConfig {
    @Bean
    @Fallback
    public CacheManager defaultCacheManager() {
        return new ConcurrentMapCacheManager();
    }
    // Skipped if another CacheManager bean exists
}
```

### Application Version in Info Endpoint

```yaml
# Automatic application version from manifest
management:
  info:
    java:
      enabled: true
    os:
      enabled: true
```

### Migration Notes

- Structured logging replaces custom Logback/Log4j2 JSON layouts for common formats
- `@Fallback` is more explicit than `@ConditionalOnMissingBean` for library authors
- Review existing `@ConditionalOnMissingBean` patterns — `@Fallback` may be cleaner

---

## 7. Spring Boot 3.4 → 4.0 (GA: November 2025)

### Jakarta EE 11 Baseline

- **Servlet 6.1** (Tomcat 11.0), **JPA 3.2**, **Bean Validation 3.1**
- Requires Jakarta EE 11 compatible runtime — Tomcat 10.x is NOT compatible
- Hibernate ORM 7, Hibernate Validator 9

### Jackson 3.0 Default

```java
// Jackson 3 is now the default JSON processor
// Jackson 2 remains available via spring-boot-jackson2 (deprecated)
// Key differences:
// - Stricter defaults (e.g., fail on unknown properties by default)
// - Improved immutability support
// - Some annotation behavior changes
```

- Jersey 4.0 does not yet support Jackson 3 — use `spring-boot-jackson2` for Jersey
- Review custom `ObjectMapper` configuration for Jackson 3 compatibility

### Built-in Resilience (Replaces spring-retry)

```java
// Boot 4.0: native resilience via spring-boot-starter-resilience
@Service
@EnableResilientMethods
public class PaymentService {

    @Retryable(maxAttempts = 3, delay = 100)
    public PaymentResult processPayment(String paymentId) {
        return paymentGateway.charge(paymentId);
    }

    @ConcurrencyLimit(2) // Bulkhead pattern — limits concurrent executions
    public Report generateReport(Long reportId) {
        return reportGenerator.generate(reportId);
    }
}
```

- `spring-retry` library replaced by built-in retry in `spring-core`
- Add `spring-boot-starter-resilience` for auto-configuration
- `@Retryable` and `@ConcurrencyLimit` are framework-native annotations

### JSpecify Null Safety

```java
// Boot 4.0: JSpecify annotations replace Spring @Nullable/@NonNull
import org.jspecify.annotations.Nullable;
import org.jspecify.annotations.NonNull;

// Spring's own @Nullable/@NonNull (JSR-305 based) are deprecated
// Migrate: org.springframework.lang.Nullable → org.jspecify.annotations.Nullable
```

### API Versioning

```java
// Boot 4.0: auto-configured API versioning support
@RestController
@RequestMapping("/api/users")
@ApiVersion("2")
public class UserControllerV2 {
    @GetMapping("/{id}")
    public UserResponseV2 getUser(@PathVariable Long id) { ... }
}
```

- Framework-managed versioning via `ApiVersionStrategy`
- Supports header-based, URL path, and content-negotiation strategies

### HTTP Interface Client Auto-Configuration

```java
// Boot 4.0: HTTP Interface Clients auto-configured from properties
@HttpExchange("/api/users")
public interface UserClient {
    @GetExchange("/{id}")
    User getUser(@PathVariable Long id);

    @PostExchange
    User createUser(@RequestBody CreateUserRequest request);
}
```

### Starter Module Changes

| Boot 3.x                             | Boot 4.0                        |
| ------------------------------------- | ------------------------------- |
| Jackson included in web starter       | `spring-boot-jackson` (Jackson 3) |
| `spring-retry` external dependency    | `spring-boot-starter-resilience` |
| Manual HTTP Interface Client setup    | Auto-configured from properties |

### Other Key Upgrades

- **Spring Security 7.0** (Authorization Server moved into Security)
- **Kotlin 2.2**, **JUnit 6.0** (JUnit Platform 2.0)
- **Micrometer 1.16**, **Reactor 2025.0**
- **Flyway 11**, **Liquibase 5.0**
- **Gradle 9** recommended for build
- `spring-jcl` removed — ensure SLF4J is on classpath

### Migration Notes

- **Jakarta EE 11 is mandatory** — update Servlet container to Tomcat 11 / Jetty 12.1
- Replace `spring-retry` with native `@Retryable` / `@ConcurrencyLimit`
- Review Jackson configuration for Jackson 3.0 compatibility
- Replace JSR-305 / Spring `@Nullable` with JSpecify annotations
- Test Hibernate ORM 7 behavior — ID generation and query semantics may differ
- Check starter name changes — some starters have been renamed or split
- Use OpenRewrite `UpgradeSpringBoot_4_0` recipe for automated migration
- `spring-jcl` removal: verify SLF4J is on classpath
- Review Spring Security 7.0 changes if using OAuth2 or Authorization Server

---

## 8. Migration Checklist

### Before Upgrade

- [ ] Verify Java version meets minimum requirement
- [ ] Run `spring-boot-properties-migrator` (for 2.x → 3.x)
- [ ] Check dependency compatibility (especially starters and Hibernate version)
- [ ] Review deprecated API warnings in build output
- [ ] Backup database and configuration before production upgrade

### Upgrade Procedure

1. Update Spring Boot version in `build.gradle.kts` or `pom.xml`
2. For 2.x → 3.x: run javax → jakarta migration
3. Clean build — fix compile errors
4. Fix property name changes (check startup warnings)
5. Fix Security configuration changes
6. Run full test suite
7. Validate in staging environment

### After Upgrade

- [ ] Remove `spring-boot-properties-migrator` dependency
- [ ] Adopt new features (virtual threads, RestClient, structured logging, etc.)
- [ ] Replace deprecated APIs
- [ ] Update Docker base images if Java version changed
- [ ] Update CI pipeline configuration

---

## 9. Version Compatibility Matrix

### Spring Boot ↔ Spring Framework ↔ Hibernate

| Boot    | Framework | Hibernate | Jakarta EE | Tomcat  |
| ------- | --------- | --------- | ---------- | ------- |
| 2.7.x   | 5.3.x     | 5.6.x     | javax      | 9.0.x   |
| 3.0.x   | 6.0.x     | 6.1.x     | jakarta    | 10.1.x  |
| 3.1.x   | 6.0.x     | 6.2.x     | jakarta    | 10.1.x  |
| 3.2.x   | 6.1.x     | 6.4.x     | jakarta    | 10.1.x  |
| 3.3.x   | 6.1.x     | 6.5.x     | jakarta    | 10.1.x  |
| 3.4.x   | 6.2.x     | 6.6.x     | jakarta    | 10.1.x  |
| 4.0.x   | 7.0.x     | 7.0.x     | jakarta EE 11 | 11.0.x |

### Spring Boot ↔ Spring Security

| Boot    | Spring Security | Key Change                          |
| ------- | --------------- | ----------------------------------- |
| 2.7.x   | 5.7–5.8         | WebSecurityConfigurerAdapter deprecated |
| 3.0.x   | 6.0             | Adapter removed, lambda DSL default |
| 3.1.x   | 6.1             | Authorization improvements          |
| 3.2.x   | 6.2             | Passkeys support                    |
| 3.3.x   | 6.3             | Improved OAuth2                     |
| 3.4.x   | 6.4             | One-time token login                |
| 4.0.x   | 7.0             | Authorization Server merged, OAuth2 improvements |
