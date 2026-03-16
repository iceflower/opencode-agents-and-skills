---
name: spring-framework-migration
description: >-
  Spring Framework version migration guide covering 5.x to 7.0.
  Use when upgrading Spring Framework versions, migrating javax to jakarta,
  or adopting RestClient, JdbcClient, or HTTP Interface Clients.
---

# Spring Framework Migration Rules

## 1. Version Overview

| Version | Release    | Key Theme                                            |
| ------- | ---------- | ---------------------------------------------------- |
| 5.0     | 2017-09    | Java 8 baseline, Kotlin support, WebFlux             |
| 5.1     | 2018-09    | Functional bean registration, Java 11 support        |
| 5.2     | 2019-09    | RSocket, Coroutines support, Java 13                 |
| 5.3     | 2020-10    | Java 15, GraalVM native hints, PathPattern parser    |
| 6.0     | 2022-11    | Java 17 baseline, Jakarta EE 9+, AOT, Observation   |
| 6.1     | 2023-11    | RestClient, JdbcClient, virtual thread readiness     |
| 6.2     | 2024-11    | Bean overriding control, fallback beans              |
| 7.0     | 2025-11    | Jakarta EE 11, JSpecify null safety, built-in retry  |

---

## 2. Spring Framework 5.x → 6.0

### Java 17 Minimum Baseline

- Java 8/11 no longer supported — update JDK before upgrading
- Enables use of records, sealed classes, pattern matching in framework internals

### Jakarta EE 9+ Migration (Most Impactful Change)

```java
// Before (javax namespace)
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.servlet.http.HttpServletRequest;
import javax.validation.constraints.NotNull;
import javax.annotation.PostConstruct;

// After (jakarta namespace)
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.validation.constraints.NotNull;
import jakarta.annotation.PostConstruct;
```

### Full javax → jakarta Mapping

| javax Package              | jakarta Package              | Affected Area        |
| -------------------------- | ---------------------------- | -------------------- |
| `javax.persistence.*`      | `jakarta.persistence.*`      | JPA entities         |
| `javax.servlet.*`          | `jakarta.servlet.*`          | Filters, servlets    |
| `javax.validation.*`       | `jakarta.validation.*`       | Bean Validation      |
| `javax.annotation.*`       | `jakarta.annotation.*`       | @PostConstruct, etc. |
| `javax.transaction.*`      | `jakarta.transaction.*`      | JTA transactions     |
| `javax.mail.*`             | `jakarta.mail.*`             | Email                |
| `javax.websocket.*`        | `jakarta.websocket.*`        | WebSocket            |
| `javax.inject.*`           | `jakarta.inject.*`           | CDI injection        |

### Migration Strategy

1. Use IDE global search-replace: `javax.persistence` → `jakarta.persistence` (etc.)
2. Update dependencies: `javax.persistence:javax.persistence-api` → `jakarta.persistence:jakarta.persistence-api`
3. Hibernate 6.x is required (Hibernate 5.x uses javax)
4. Tomcat 10.1+ / Jetty 12+ required for Jakarta Servlet API

### Removed APIs in 6.0

| Removed                                     | Replacement                              |
| ------------------------------------------- | ---------------------------------------- |
| `AsyncRestTemplate`                         | `WebClient` or `RestClient` (6.1+)       |
| `RestTemplate` (not removed, but discouraged) | `RestClient` (6.1+) for new code       |
| `CommonsMultipartResolver`                  | `StandardServletMultipartResolver`       |
| `org.springframework.util.ClassUtils` (some methods) | Standard Java reflection         |
| `MediaType.APPLICATION_JSON_UTF8`           | `MediaType.APPLICATION_JSON`             |

### AOT (Ahead-of-Time) Processing

- Spring Framework 6 introduces AOT processing for GraalVM native images
- `@RegisterReflectionForBinding` for types that need reflection at runtime
- Proxy classes generated at build time instead of runtime
- Conditional beans must be resolvable at build time for AOT

### Observation API (Micrometer Integration)

```java
// Framework 6 integrates Micrometer Observation API natively
// Automatic instrumentation for:
// - Spring MVC (@Controller methods)
// - RestTemplate / RestClient
// - WebClient
// - Spring Security
// - Spring Data repositories (6.1+)
```

### Migration Notes

- **javax → jakarta is a breaking change** — all import statements must be updated
- No backward compatibility — cannot mix javax and jakarta in the same project
- Use `IntelliJ IDEA` migration tool or `OpenRewrite` recipes for automated migration
- Some third-party libraries may not support Jakarta EE 9+ yet — verify compatibility

---

## 3. Spring Framework 6.0 → 6.1

### RestClient (New)

```java
// RestClient — synchronous HTTP client replacing RestTemplate
RestClient restClient = RestClient.builder()
    .baseUrl("https://api.example.com")
    .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
    .build();

User user = restClient.get()
    .uri("/users/{id}", userId)
    .retrieve()
    .body(User.class);
```

### JdbcClient (New)

```java
// JdbcClient — fluent API for JDBC operations
JdbcClient jdbcClient = JdbcClient.create(dataSource);

Optional<User> user = jdbcClient.sql("SELECT * FROM users WHERE id = ?")
    .param(userId)
    .query(User.class)
    .optional();

List<User> users = jdbcClient.sql("SELECT * FROM users WHERE status = :status")
    .param("status", "ACTIVE")
    .query(User.class)
    .list();
```

### Virtual Thread Readiness

- Framework internals updated to avoid `synchronized` pinning
- `SimpleAsyncTaskExecutor` supports virtual threads
- `@Async` compatible with virtual thread executors

### HTTP Interface Clients (Stable)

```java
// Declarative HTTP client using interfaces
@HttpExchange("/api/users")
public interface UserClient {

    @GetExchange("/{id}")
    User getUser(@PathVariable Long id);

    @PostExchange
    User createUser(@RequestBody CreateUserRequest request);

    @GetExchange
    List<User> listUsers(@RequestParam("status") String status);
}

// Registration
@Bean
public UserClient userClient(RestClient.Builder builder) {
    RestClient restClient = builder.baseUrl("https://api.example.com").build();
    RestClientAdapter adapter = RestClientAdapter.create(restClient);
    HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(adapter).build();
    return factory.createClient(UserClient.class);
}
```

### Migration Notes

- `RestTemplate` still works but `RestClient` is recommended for new code
- `JdbcClient` does not replace Spring Data JPA — use for simple JDBC queries
- HTTP Interface Clients provide Feign-like declarative HTTP without external dependency

---

## 4. Spring Framework 6.1 → 6.2

### Bean Overriding Control

```java
// 6.2+: control whether bean overriding is allowed
@Configuration
public class AppConfig {
    // By default, bean overriding now produces a warning
    // Set spring.main.allow-bean-definition-overriding=false to make it an error
}
```

### Fallback Beans

```java
// 6.2+: @Fallback annotation for default bean definitions
@Bean
@Fallback
public CacheManager defaultCacheManager() {
    return new ConcurrentMapCacheManager();
}

// If another CacheManager bean exists, the @Fallback bean is skipped
// If no other CacheManager exists, the @Fallback bean is used
```

### Migration Notes

- Review bean overriding in tests — may need explicit `@TestConfiguration` or `@Primary`
- `@Fallback` replaces common `@ConditionalOnMissingBean` patterns for framework code

---

## 5. Spring Framework 6.x → 7.0 (GA: November 2025)

### Jakarta EE 11 Baseline

- **Servlet 6.1** (Tomcat 11.0), **JPA 3.2**, **Bean Validation 3.1**
- Jakarta EE 10 servers are NOT compatible — requires EE 11 runtime
- Hibernate ORM 7, Hibernate Validator 9

### JSpecify Null Safety (Replaces JSR-305)

```java
// Before (Framework 6.x): JSR-305 annotations
import org.springframework.lang.Nullable;
import org.springframework.lang.NonNull;

// After (Framework 7.0): JSpecify annotations
import org.jspecify.annotations.Nullable;
import org.jspecify.annotations.NonNull;
```

- Spring's own `@Nullable` / `@NonNull` annotations deprecated in favor of JSpecify
- JSpecify supports nullness on generic types, arrays, and vararg elements
- Better tooling integration (NullAway, IntelliJ IDEA, Kotlin interop)
- Use `@NullMarked` on packages for null-safe-by-default

### Built-in Resilience (Framework Native)

Framework 7.0 includes native retry and concurrency limiting in `spring-core`:

```java
@Service
@EnableResilientMethods
public class PaymentService {

    @Retryable(maxAttempts = 3, delay = 100)
    public PaymentResult processPayment(String paymentId) {
        return paymentGateway.charge(paymentId);
    }

    @ConcurrencyLimit(2) // Bulkhead pattern
    public Report generateReport(Long reportId) {
        return reportGenerator.generate(reportId);
    }
}
```

- `@Retryable` and `@ConcurrencyLimit` are framework-native annotations
- Adopted by Spring AMQP, Kafka, Integration, and Batch
- See `spring-boot-migration` skill for Boot 4.0 auto-configuration details

### API Versioning (Framework Native)

Framework 7.0 supports declarative API versioning:

```java
@RestController
@RequestMapping("/api/users")
@ApiVersion("2")
public class UserControllerV2 {
    @GetMapping("/{id}")
    public UserResponseV2 getUser(@PathVariable Long id) { ... }
}
```

- Framework-managed versioning via `ApiVersionStrategy`
- Supports header-based, URL-based, and content-negotiation versioning
- See `api-versioning` skill for detailed compatibility principles

### Jackson 3.0 Support

- Jackson 3.0 is the default JSON processor
- Jackson 2.x remains available in deprecated form
- Review custom `ObjectMapper` configuration for compatibility

### Other Key Changes

- **Kotlin 2.2** support (from Kotlin 2.x baseline)
- **JUnit 6.0** (JUnit Platform 2.0)
- **GraalVM 25** support
- `spring-jcl` module removed — use SLF4J directly
- Programmatic AOT improvements for native images

### Migration Notes

- **Jakarta EE 11 is mandatory** — update Servlet container (Tomcat 11, Jetty 12.1)
- Replace JSR-305 / Spring `@Nullable` with JSpecify annotations
- Replace `spring-retry` dependency with native `@Retryable` / `@ConcurrencyLimit`
- Review Jackson customization for Jackson 3.0 compatibility
- Test with Hibernate ORM 7 — query behavior and ID generation may differ
- `spring-jcl` removal: ensure SLF4J is on classpath
- Verify Kotlin version is 2.2+ if using Kotlin

### See Also

For Spring Boot 4.0 specific auto-configuration and starter changes, see `spring-boot-migration` skill.

---

## 6. Migration Checklist

### Before Upgrade

- [ ] Verify current Java version meets minimum requirement
- [ ] Check all dependencies for Jakarta EE namespace support (for 5.x → 6.x)
- [ ] Review deprecated API usage in codebase
- [ ] Verify build tool plugin versions (Maven/Gradle Spring plugins)
- [ ] Check Servlet container version compatibility

### Upgrade Procedure

1. Update Spring Framework version in build configuration
2. For 5.x → 6.x: run javax → jakarta migration (IDE or OpenRewrite)
3. Clean build — fix compile errors
4. Fix deprecation warnings
5. Run full test suite
6. Test with target Servlet container version

### After Upgrade

- [ ] Adopt new APIs (`RestClient`, `JdbcClient`, HTTP Interface Clients)
- [ ] Replace deprecated APIs before next major version
- [ ] Review AOT compatibility if targeting GraalVM native images
- [ ] Update CI pipeline with new minimum Java version
