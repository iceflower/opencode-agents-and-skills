# Dockerfile Rules

## 1. Multi-Stage Build Pattern

Separate build dependencies from runtime to minimize image size and attack surface.

```dockerfile
# Stage 1: Build
FROM builder-image AS build
WORKDIR /app
COPY dependency-files ./
RUN install-dependencies
COPY src ./src
RUN build-command

# Stage 2: Runtime
FROM runtime-image AS runtime
WORKDIR /app
COPY --from=build /app/output ./
ENTRYPOINT ["./app"]
```

---

## 2. Base Image Selection Criteria

- Use minimal runtime images (not build images) in production
- Prefer Alpine/slim/distroless variants for smaller image size
- Always pin to a specific tag — never use `latest`
- Choose images with active security maintenance
- Use distroless when shell access is not needed

---

## 3. Layer Caching Optimization

### Copy Order (least changing → most changing)

1. Build tool config files (rarely changes)
2. Dependency lock files → install dependencies (changes on dep update)
3. Source code (changes frequently)
4. Build command

### Cache Rules

- Separate dependency download from source compilation
- Copy only necessary files at each layer
- Avoid `COPY . .` — it invalidates cache on any file change
- Use `.dockerignore` to exclude unnecessary files

---

## 4. .dockerignore

### Required Entries

```text
.git
.gitignore
build
out
node_modules
*.md
docker-compose*.yml
Dockerfile*
.env*
.idea
*.iml
*.log
```

---

## 5. Security Best Practices

### Non-Root User

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

### Minimal Permissions

- Use read-only filesystem where possible
- Do not install unnecessary packages in runtime stage
- Remove build tools and caches from final image
- Never copy secrets into the image — use runtime environment variables

### Image Scanning

- Scan images for vulnerabilities before pushing (Trivy, Snyk)
- Update base images regularly for security patches
- Pin dependency versions for reproducible builds

---

## 6. JVM/Spring Boot Multi-Stage Build

### Standard Pattern

```dockerfile
# Stage 1: Build
FROM gradle:8-jdk21 AS build
WORKDIR /app
COPY build.gradle.kts settings.gradle.kts ./
COPY gradle ./gradle
RUN gradle dependencies --no-daemon
COPY src ./src
RUN gradle bootJar --no-daemon -x test

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=build /app/build/libs/*.jar app.jar
USER appuser
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Multi-Module Project Pattern

```dockerfile
FROM gradle:8-jdk21 AS build
WORKDIR /app
COPY build.gradle.kts settings.gradle.kts ./
COPY gradle ./gradle
COPY buildSrc ./buildSrc
COPY modules/app/build.gradle.kts modules/app/
COPY modules/domain/build.gradle.kts modules/domain/
RUN gradle dependencies --no-daemon
COPY modules ./modules
RUN gradle :modules:app:bootJar --no-daemon -x test

FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=build /app/modules/app/build/libs/*.jar app.jar
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 7. JVM Base Image Selection

| Image                               | Size   | Use Case                         |
| ----------------------------------- | ------ | -------------------------------- |
| `eclipse-temurin:21-jre-alpine`     | ~100MB | General production (recommended) |
| `gcr.io/distroless/java21-debian12` | ~100MB | Maximum security (no shell)      |
| `eclipse-temurin:21-jre`            | ~250MB | When Alpine compatibility issues |
| `eclipse-temurin:21-jdk`            | ~450MB | Build stage only                 |

### Selection Criteria

- Production: Use JRE, not JDK
- Prefer Alpine variants for smaller image size
- Use distroless when shell access is not needed
- Always pin to specific tag, never use `latest`

---

## 8. Gradle Layer Caching

```dockerfile
# 1. Build tool config (rarely changes)
COPY build.gradle.kts settings.gradle.kts ./
COPY gradle ./gradle

# 2. Download dependencies (changes on dependency update)
RUN gradle dependencies --no-daemon

# 3. Source code (changes frequently)
COPY src ./src

# 4. Build
RUN gradle bootJar --no-daemon -x test
```

---

## 9. JVM Runtime Options

### Production Defaults

```dockerfile
ENTRYPOINT ["java", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:InitialRAMPercentage=50.0", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

### Container-Aware Settings

| Option                    | Purpose                                    |
| ------------------------- | ------------------------------------------ |
| `MaxRAMPercentage=75`     | Use up to 75% of container memory for heap |
| `InitialRAMPercentage=50` | Start with 50% heap allocation             |

- `UseContainerSupport` is enabled by default since JDK 11 — no need to set explicitly

### JVM Memory Rules

- Always leave 25%+ memory for non-heap (metaspace, thread stacks, GC)
- Do not set fixed `-Xmx` — use percentage-based options for container flexibility
- Use `JAVA_OPTS` or `JAVA_TOOL_OPTIONS` environment variable for runtime overrides

---

## 10. Anti-Patterns

- Using `latest` tag for base images
- Running as root in production
- Copying entire project with `COPY . .`
- Installing build tools in runtime image
- Hardcoding secrets or config values in Dockerfile
- Skipping `.dockerignore`
- Single-stage build including build tools in final image
- Using `ADD` when `COPY` suffices (`ADD` has extra behavior: URL fetch, tar extraction)
