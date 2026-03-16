---
name: spring-troubleshooting
description: >-
  Spring Boot troubleshooting including startup failures, JVM OOM diagnosis,
  HikariCP issues, and Spring-specific slow API debugging.
  Use when diagnosing Spring Boot application issues.
---

# Spring Boot Troubleshooting Rules

## 1. Spring Boot Startup Failure

### Diagnosis Order

1. Read the full stack trace — the root cause is usually at the bottom
2. Check for `BeanCreationException` → missing or conflicting beans
3. Check for `DataSourceAutoConfiguration` → database connection issue
4. Check for port conflict → `PortInUseException`
5. Check active profile → verify `spring.profiles.active` is set correctly

### Common Causes and Fixes

| Error                                         | Cause                          | Fix                                                 |
| --------------------------------------------- | ------------------------------ | --------------------------------------------------- |
| `UnsatisfiedDependencyException`              | Missing bean or circular dep   | Check `@Component` scan path, break circular dep    |
| `HikariPool-1 - Connection is not available`  | DB unreachable or creds wrong  | Verify `spring.datasource.*` config and network     |
| `Port already in use`                         | Another process on same port   | Kill the process or change `server.port`            |
| `NoSuchBeanDefinitionException`               | Bean not registered            | Check package scan, `@Configuration` class          |
| `Failed to configure a DataSource`            | Missing DB driver or URL       | Add `runtimeOnly` driver dep, set datasource URL    |

---

## 2. JVM OOM (OutOfMemoryError)

### OOM Types

1. `Java heap space` → heap exhaustion
2. `Metaspace` → too many classes loaded
3. `GC overhead limit exceeded` → spending 98%+ time in GC
4. `unable to create new native thread` → thread limit reached

### Investigation Commands

```bash
# Check current JVM memory settings inside container
jcmd 1 VM.flags

# Generate heap dump on OOM (add to ENTRYPOINT)
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heapdump.hprof

# Force heap dump from running process
jcmd 1 GC.heap_dump /tmp/heapdump.hprof

# Analyze heap dump with MAT or VisualVM
```

---

## 3. Spring-Specific Slow API Response

### Profiling Approach

```kotlin
val startTime = System.currentTimeMillis()
val result = someOperation()
val duration = System.currentTimeMillis() - startTime
log.info("Operation completed", mapOf("durationMs" to duration))
```

### Spring-Specific Causes and Fixes

| Cause                                     | Diagnosis                    | Fix                                |
| ----------------------------------------- | ---------------------------- | ---------------------------------- |
| N+1 queries                               | Multiple similar SQL in logs | Use `JOIN FETCH` or `@EntityGraph` |
| JPA query logging off                     | Cannot see SQL               | Enable `spring.jpa.show-sql=true`  |
| No caching                                | Same query repeated          | Add `@Cacheable` or Redis cache    |
| HikariCP pool exhaustion                  | Pool warning in logs         | Increase pool size or optimize     |
| Missing `@Transactional(readOnly = true)` | Unnecessary write locks      | Add readOnly for read operations   |

---

## Related Skills

- **troubleshooting** — Framework-agnostic debugging patterns (slow APIs, deployment rollback, connection issues)
