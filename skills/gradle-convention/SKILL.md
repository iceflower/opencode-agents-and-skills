---
name: gradle-convention
description: Gradle build conventions for Kotlin/Spring Boot multi-module projects.
  Use when writing or reviewing build.gradle.kts, settings.gradle.kts, or version
  catalog files.
---

# Gradle Convention Rules

## 1. Multi-Module Project Structure

### Recommended Layout

```text
project-root/
├── settings.gradle.kts
├── build.gradle.kts              # Root: shared config
├── buildSrc/                     # Convention plugins
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│       └── kotlin-conventions.gradle.kts
├── gradle/
│   └── libs.versions.toml        # Version catalog
└── modules/
    ├── app/                      # Spring Boot application
    │   ├── build.gradle.kts
    │   └── src/
    ├── domain/                   # Domain logic
    │   ├── build.gradle.kts
    │   └── src/
    └── infrastructure/           # External integrations
        ├── build.gradle.kts
        └── src/
```

### Module Dependency Direction

```text
app → domain ← infrastructure
```

- `domain`: Pure business logic, no framework dependencies
- `app`: Spring Boot application, controllers, configuration
- `infrastructure`: Database, external APIs, messaging
- `domain` should never depend on `app` or `infrastructure`

---

## 2. Version Catalog

### libs.versions.toml

```toml
[versions]
spring-boot = "4.0.3"
kotlin = "2.3.10"
kotest = "5.9.0"
mockk = "1.13.13"

[libraries]
spring-boot-starter-web = { module = "org.springframework.boot:spring-boot-starter-web" }
spring-boot-starter-data-jpa = { module = "org.springframework.boot:spring-boot-starter-data-jpa" }
spring-boot-starter-test = { module = "org.springframework.boot:spring-boot-starter-test" }
kotest-runner = { module = "io.kotest:kotest-runner-junit5", version.ref = "kotest" }
kotest-spring = { module = "io.kotest.extensions:kotest-extensions-spring", version = "1.3.0" }
mockk = { module = "io.mockk:mockk", version.ref = "mockk" }

[bundles]
kotest = ["kotest-runner", "kotest-spring", "mockk"]

[plugins]
spring-boot = { id = "org.springframework.boot", version.ref = "spring-boot" }
kotlin-jvm = { id = "org.jetbrains.kotlin.jvm", version.ref = "kotlin" }
kotlin-spring = { id = "org.jetbrains.kotlin.plugin.spring", version.ref = "kotlin" }
kotlin-jpa = { id = "org.jetbrains.kotlin.plugin.jpa", version.ref = "kotlin" }
```

### Usage in build.gradle.kts

```kotlin
dependencies {
    implementation(libs.spring.boot.starter.web)
    implementation(libs.spring.boot.starter.data.jpa)
    testImplementation(libs.bundles.kotest)
}
```

### Version Catalog Rules

- All dependency versions must be defined in `libs.versions.toml`
- Never hardcode version strings in `build.gradle.kts`
- Use `bundles` to group related test/utility dependencies
- Keep versions up to date — check for updates regularly

---

## 3. Dependency Declarations

### Configuration Types

| Configuration | Purpose | Transitive |
| --- | --- | --- |
| `implementation` | Internal dependency | No |
| `api` | Exposed to consumers | Yes |
| `compileOnly` | Compile-time only (annotations, etc.) | No |
| `runtimeOnly` | Runtime only (JDBC drivers, etc.) | No |
| `testImplementation` | Test dependencies | No |

### Rules

```kotlin
dependencies {
    // Use implementation by default
    implementation(libs.spring.boot.starter.web)

    // Use api only in library modules when the type is part of the public API
    api(libs.some.shared.model)

    // Use compileOnly for annotation processors
    compileOnly(libs.lombok)

    // Use runtimeOnly for runtime-only dependencies
    runtimeOnly(libs.postgresql)

    // Test dependencies
    testImplementation(libs.bundles.kotest)
    testImplementation(libs.spring.boot.starter.test)
}
```

- Default to `implementation` — only use `api` when the dependency type appears in public signatures
- Use `runtimeOnly` for JDBC drivers, logging backends
- Use `compileOnly` for compile-time annotations

---

## 4. Convention Plugins (buildSrc)

### Shared Configuration

```kotlin
// buildSrc/src/main/kotlin/kotlin-conventions.gradle.kts
plugins {
    kotlin("jvm")
}

group = "com.example"

kotlin {
    jvmToolchain(21)
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

### Spring Boot Module Convention

```kotlin
// buildSrc/src/main/kotlin/spring-boot-conventions.gradle.kts
plugins {
    id("kotlin-conventions")
    id("org.springframework.boot")
    kotlin("plugin.spring")
}

// Spring Boot plugin applies BOM — no version needed for starters
dependencies {
    implementation(platform(org.springframework.boot.gradle.plugin.SpringBootPlugin.BOM_COORDINATES))
    implementation("org.springframework.boot:spring-boot-starter")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

### Convention Plugin Rules

- Extract common build logic into `buildSrc` convention plugins
- Apply convention plugins in module `build.gradle.kts` instead of repeating config
- Keep convention plugins focused — one per concern (kotlin, spring, jpa)

---

## 5. Build Optimization

### gradle.properties

```properties
# Parallel execution
org.gradle.parallel=true

# Build cache
org.gradle.caching=true

# Daemon (keep JVM alive between builds)
org.gradle.daemon=true

# JVM memory for Gradle daemon
org.gradle.jvmargs=-Xmx2g -XX:+UseParallelGC

# Kotlin incremental compilation
kotlin.incremental=true
```

### CI-Specific Settings

```properties
# In CI: disable daemon (short-lived environments)
org.gradle.daemon=false
```

---

## 6. Task Conventions

### Custom Task Naming

- Use camelCase for task names
- Prefix with action verb: `generate`, `check`, `publish`
- Group related tasks with `group` property

### Common Tasks

| Task | Purpose |
| --- | --- |
| `./gradlew build` | Compile + test + assemble |
| `./gradlew bootJar` | Build Spring Boot executable JAR |
| `./gradlew test` | Run all tests |
| `./gradlew dependencies` | Show dependency tree |
| `./gradlew dependencyUpdates` | Check for dependency updates |

---

## 7. Anti-Patterns

- Hardcoding dependency versions in `build.gradle.kts`
- Using `compile` (deprecated) instead of `implementation`
- Applying plugins in `allprojects`/`subprojects` blocks (use convention plugins)
- Copying build logic across module `build.gradle.kts` files
- Using `buildscript` block when plugin DSL is available
- Skipping `gradle wrapper` — always commit the wrapper
