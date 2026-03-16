---
name: spring-config
description: Spring Boot configuration management patterns. Use when writing or
  reviewing application.yml, profile config, @ConfigurationProperties, or actuator
  settings.
---

# Spring Boot Configuration Rules

## 1. Profile Management

### Profile Structure

```text
src/main/resources/
├── application.yml              # Common settings (all profiles)
├── application-local.yml        # Local development
├── application-dev.yml          # Dev environment
├── application-staging.yml      # Staging environment
└── application-prod.yml         # Production environment
```

### Profile Activation

```yaml
# application.yml — default profile
spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:local}
```

### Profile Separation Rules

| Setting              | Common | Local | Dev | Staging | Prod |
| -------------------- | ------ | ----- | --- | ------- | ---- |
| Server port          | Yes    |       |     |         |      |
| DB URL               |        | Yes   | Yes | Yes     | Yes  |
| Log level            |        | Yes   | Yes | Yes     | Yes  |
| Feature flags        |        |       | Yes | Yes     | Yes  |
| Connection pool size |        |       | Yes | Yes     | Yes  |
| CORS origins         |        |       | Yes | Yes     | Yes  |

- Common settings go in `application.yml`
- Environment-specific overrides go in profile files
- Never put secrets in any yml file — use environment variables

---

## 2. @ConfigurationProperties

### Preferred Pattern

```kotlin
@ConfigurationProperties(prefix = "app.feature")
data class FeatureProperties(
    val enabled: Boolean = false,
    val maxRetries: Int = 3,
    val timeout: Duration = Duration.ofSeconds(30),
    val allowedOrigins: List<String> = emptyList()
)
```

### Registration

```kotlin
@Configuration
@EnableConfigurationProperties(FeatureProperties::class)
class AppConfig
```

### Configuration Properties Rules

- Always use `@ConfigurationProperties` over `@Value` for grouped config
- Use `data class` for immutable configuration binding
- Provide sensible defaults for all properties
- Use `Duration`, `DataSize` types instead of raw numbers
- Validate with `@Validated` and JSR-303 annotations when needed

```kotlin
@Validated
@ConfigurationProperties(prefix = "app.http-client")
data class HttpClientProperties(
    val connectTimeout: Duration = Duration.ofSeconds(5),

    val readTimeout: Duration = Duration.ofSeconds(30),

    @field:Min(1) @field:Max(100)
    val maxConnections: Int = 20
)
```

---

## 3. Environment Variable Binding

### Naming Convention

```yaml
# application.yml
app:
  database:
    connection-pool-size: ${APP_DATABASE_CONNECTION_POOL_SIZE:10}
    max-lifetime: ${APP_DATABASE_MAX_LIFETIME:30m}
```

### Binding Rules

| YAML key                | Environment variable              |
| ----------------------- | --------------------------------- |
| `app.feature.enabled`   | `APP_FEATURE_ENABLED`             |
| `app.db.pool-size`      | `APP_DB_POOL_SIZE`                |
| `spring.datasource.url` | `SPRING_DATASOURCE_URL`           |

- Use `${ENV_VAR:default}` syntax for all environment-specific values
- Kebab-case in YAML, SCREAMING_SNAKE in env vars
- Always provide defaults for non-secret values
- Never provide defaults for secrets (force explicit configuration)

---

## 4. Secret Management

### Do

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

jwt:
  secret: ${JWT_SECRET}
```

### Do Not

```yaml
# Never hardcode secrets
spring:
  datasource:
    password: my-secret-password  # WRONG

# Never commit default secrets
jwt:
  secret: ${JWT_SECRET:default-secret}  # WRONG
```

### Secret Handling Rules

- Secrets must come from environment variables or secret managers
- No default values for secrets — fail fast if missing
- Use Kubernetes Secrets or cloud provider secret managers in production
- Rotate secrets without application restart when possible

---

## 5. Connection Pool Configuration

### HikariCP Defaults

```yaml
spring:
  datasource:
    hikari:
      minimum-idle: ${DB_POOL_MIN_IDLE:5}
      maximum-pool-size: ${DB_POOL_MAX_SIZE:10}
      idle-timeout: ${DB_POOL_IDLE_TIMEOUT:300000}
      max-lifetime: ${DB_POOL_MAX_LIFETIME:1800000}
      connection-timeout: ${DB_POOL_CONNECTION_TIMEOUT:30000}
      pool-name: app-hikari-pool
```

### Sizing Guidelines

| Environment | Min Idle | Max Pool | Rationale                  |
| ----------- | -------- | -------- | -------------------------- |
| Local       | 2        | 5        | Minimal resources          |
| Dev         | 3        | 10       | Moderate concurrency       |
| Staging     | 5        | 15       | Near-production load       |
| Production  | 5        | 20       | Handle peak traffic        |

- Formula: `max_pool_size = (core_count * 2) + disk_spindles`
- Always set `max-lifetime` below database connection timeout
- Monitor pool usage before adjusting sizes

---

## 6. Actuator Configuration

### Recommended Setup

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

### Security Rules

- Never expose all actuator endpoints in production
- Protect sensitive endpoints (env, configprops, beans) behind authentication
- Use separate management port for internal-only endpoints when needed

---

## 7. Anti-Patterns

- Hardcoding environment-specific values in `application.yml`
- Using `@Value` for complex or grouped configuration
- Putting secrets with default values in config files
- Duplicating common settings across profile files
- Using `spring.profiles.include` chains that are hard to trace
- Missing validation on configuration properties
- Exposing all actuator endpoints without access control
