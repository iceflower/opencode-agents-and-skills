---
name: kotlin-migration
description: >-
  Kotlin version migration guide and breaking changes reference.
  Use when upgrading Kotlin versions, replacing deprecated APIs,
  or reviewing version-specific language features (K2, value class, Enum.entries, etc.).
---

# Kotlin Version Migration Rules

## 1. Version Overview

| Version | Release    | Key Theme                                    |
| ------- | ---------- | -------------------------------------------- |
| 1.3     | 2018-10    | Coroutines stable                            |
| 1.4     | 2020-08    | SAM conversions, trailing commas             |
| 1.5     | 2021-05    | Value classes, sealed interfaces, JVM records|
| 1.6     | 2021-11    | Exhaustive when, builder inference stable    |
| 1.7     | 2022-06    | K2 alpha, min/max API changes                |
| 1.8     | 2022-12    | Java 8 minimum target, kotlin-reflect slim   |
| 1.9     | 2023-07    | K2 beta, Enum.entries, data object           |
| 2.0     | 2024-05    | K2 compiler stable, smart cast improvements  |
| 2.1     | 2024-11    | Guard conditions in when, multi-dollar strings|
| 2.2     | 2025-06    | Context parameters stable, guard/break/continue stable|
| 2.3     | 2025-12    | Explicit backing fields, unused return value checker|

---

## 2. Kotlin 1.4 → 1.5

### Value Classes (Replaces Inline Classes)

```kotlin
// 1.4: inline class (deprecated)
inline class UserId(val value: Long)

// 1.5+: value class
@JvmInline
value class UserId(val value: Long)
```

- `inline class` is deprecated — replace with `@JvmInline value class`
- `@JvmInline` is required on JVM backend

### Sealed Interfaces

```kotlin
// 1.5+: sealed interface support
sealed interface Result<out T>
data class Success<T>(val data: T) : Result<T>
data class Failure(val error: Throwable) : Result<Nothing>
```

- Unlike `sealed class`, sealed interfaces allow multiple inheritance
- Prefer sealed interface when designing type hierarchies

### Stable Unsigned Integer Types

```kotlin
val size: UInt = 42u
val bigId: ULong = 123456789uL
```

### Migration Notes

- Bulk replacing `inline class` → `value class` may break binary compatibility
- Adding `@JvmInline` to library modules changes ABI — coordinate with consumers

---

## 3. Kotlin 1.5 → 1.6

### Exhaustive When Statements

```kotlin
sealed class Status { ... }

// 1.6+: warning when not all cases are covered in when-statement
// 1.7+: warning promoted to error
val label = when (status) {
    is Status.Active -> "active"
    is Status.Inactive -> "inactive"
    // missing branch causes compile error
}
```

- Previously required only for when-expressions; now also applies to when-statements
- Adding new sealed subtypes requires updating all `when` blocks

### Stable Builder Inference

```kotlin
// Type inference works reliably inside builder lambdas
buildList {
    add("hello")
    add("world")
} // Inferred as List<String>
```

### Suspend Conversions

```kotlin
// Regular functions can be passed where suspend function type is expected
fun normalFunction(): String = "result"

suspend fun process(block: suspend () -> String) = block()

// 1.6+: direct passing allowed
process(::normalFunction)
```

### Migration Notes

- Existing `when` statements without `else` branch will produce warnings — add `else` or cover all cases
- Compiler option `-Xopt-in` renamed to `-opt-in`

---

## 4. Kotlin 1.6 → 1.7

### min/max Function Behavior Change

```kotlin
// 1.6: NoSuchElementException on empty collection
listOf<Int>().min() // throws

// 1.7+: prefer minOrNull(), maxOrNull()
// min(), max() now return non-null (throw on empty collection)
listOf<Int>().minOrNull() // returns null
```

### Definitely Non-Nullable Types

```kotlin
// Improved interop with Java generics
fun <T : Any> process(value: T & Any) { ... }
```

### Opt-In Requirement Changes

```kotlin
// @RequiresOptIn annotation is now stable
@RequiresOptIn(message = "This API is experimental")
@Retention(AnnotationRetention.BINARY)
annotation class ExperimentalApi
```

### Migration Notes

- Review `min()` / `max()` usage — switch to `minOrNull()` / `maxOrNull()` if empty collection is possible
- Some `@ExperimentalStdlibApi` APIs promoted to stable

---

## 5. Kotlin 1.7 → 1.8

### JVM Target Minimum Version 1.8

```kotlin
// build.gradle.kts
kotlin {
    jvmToolchain(17) // or 21
}

// jvmTarget = "1.6" removed in 1.8
```

- `jvmTarget = "1.6"` and `"1.7"` are no longer supported
- Minimum target is `"1.8"` (Java 8)

### kotlin-reflect Size Reduction

- `kotlin-reflect` reduced from ~700KB to ~400KB
- `javax.xml` dependency removed from `kotlin-reflect`

### Stable Recursive Copy for Directories

```kotlin
// Directory recursive copy is now stable
sourcePath.copyToRecursively(targetPath, followLinks = false)
```

### Migration Notes

- Build fails if `jvmTarget` is `"1.6"` or `"1.7"` — change to `"1.8"` or higher
- Prefer `kotlin.jvmToolchain()` over `kotlinOptions.jvmTarget` in Gradle

---

## 6. Kotlin 1.8 → 1.9

### Enum.entries (Replaces values())

```kotlin
enum class Color { RED, GREEN, BLUE }

// Before 1.9: values() — allocates a new array on every call
Color.values().forEach { ... }

// 1.9+: entries — pre-allocated immutable list
Color.entries.forEach { ... }
```

- `values()` copies the array on every call — performance overhead
- `entries` returns `List<E>` with no reallocation

### Data Objects

```kotlin
// 1.9+: data object — auto-generated toString()
sealed interface Result {
    data class Success(val value: String) : Result
    data object Loading : Result  // toString() = "Loading"
    data object Error : Result    // toString() = "Error"
}
```

- `object` toString() returns `package.ClassName@hashCode`
- `data object` toString() returns just the class name — better for logging and debugging

### ..< (Open-Ended Range) Operator

```kotlin
// 1.9+: ..< operator replaces until
for (i in 0..<list.size) { ... }

// Equivalent: for (i in 0 until list.size)
```

### @Volatile for Kotlin/Native

- `@Volatile` annotation now supported on Kotlin/Native

### Migration Notes

- Bulk replace `values()` → `entries` recommended (performance improvement)
- Singleton sealed subtypes: replace `object` → `data object`
- `until` → `..<` replacement is optional (depends on readability preference)

---

## 7. Kotlin 1.9 → 2.0 (Major Version)

### K2 Compiler (Stable)

- K2 compiler becomes the default compiler
- Up to 2x faster compilation speed
- More accurate type inference and smart casts

### Smart Cast Improvements

```kotlin
// 2.0+: smart cast preserved after variable reassignment
var result: Any = fetchData()
if (result is String) {
    // Before: smart cast failed if reassignment was possible
    // 2.0: compiler analyzes more accurately
    println(result.length)
}

// 2.0+: intersection types for local variables
fun process(value: Any) {
    if (value is Comparable<*> && value is Iterable<*>) {
        // value smart-cast to Comparable & Iterable
    }
}
```

### Compiler Plugin API Changes

- K2 compiler transition changes the compiler plugin API
- Migrate from kapt to KSP (Kotlin Symbol Processing)

### Gradle Configuration Changes

```kotlin
// build.gradle.kts
// To explicitly use K1 compiler (for compatibility issues)
kotlin {
    compilerOptions {
        // Temporary workaround for K2 issues
        apiVersion.set(KotlinVersion.KOTLIN_1_9)
        languageVersion.set(KotlinVersion.KOTLIN_1_9)
    }
}
```

### Migration Notes

- **Projects using kapt**: K2 + kapt has limited support — prioritize KSP migration
- **Compile error changes**: K2 is stricter; previously passing code may produce errors
  - Especially in type inference and overload resolution
- **Gradle plugin compatibility**: `kotlin-gradle-plugin` 2.0 requires Gradle 7.6.3+ or 8.x
- Build cache is incompatible between K1 and K2 — clean build required on transition
- Some K1-only compiler options have been removed or changed

---

## 8. Kotlin 2.0 → 2.1

### Guard Conditions in When (Preview)

```kotlin
// 2.1+: guard conditions on when branches (preview — requires -Xwhen-guards)
sealed class Result {
    data class Success(val data: List<String>) : Result()
    data class Error(val code: Int) : Result()
}

fun handle(result: Result) {
    when (result) {
        is Result.Success if result.data.isNotEmpty() -> processData(result.data)
        is Result.Success -> handleEmptyResult()
        is Result.Error if result.code == 404 -> handleNotFound()
        is Result.Error -> handleGenericError(result.code)
    }
}
```

- Additional conditions can be specified with `if` after pattern matching
- Eliminates nested `if`-`else` for improved readability

### Non-Local Break and Continue (Preview)

```kotlin
// 2.1+: break/continue in inline function lambdas (preview — requires -Xnon-local-break-continue)
fun processItems(items: List<Item>) {
    for (group in groups) {
        group.forEach { item ->
            if (item.isInvalid()) continue  // 2.1+: continues outer for loop
            if (item.isTerminal()) break    // 2.1+: breaks outer for loop
            process(item)
        }
    }
}
```

### Multi-Dollar String Interpolation

```kotlin
// 2.1+: multi-dollar interpolation (experimental — requires -Xmulti-dollar-interpolation)
// $$ prefix: two dollar signs trigger interpolation, single $ is literal
val jsonSchema = $$"""
    {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "$id": "https://example.com/product.schema.json",
        "title": "$${simpleName ?: "unknown"}",
        "type": "object"
    }
"""
// $schema, $id → literal (single $)
// $${simpleName} → interpolated (double $$)
```

- `$$"..."` prefix means `$$` triggers interpolation, single `$` is treated as literal
- Reduces escaping in strings with frequent `$` literals (JSON schemas, regex, scripts)
- Requires compiler option `-Xmulti-dollar-interpolation` in Kotlin 2.1 (stable in 2.2)

### Migration Notes

- Guard conditions (preview) can replace existing `when` + inner `if` patterns — adopt after stabilization
- Non-local break/continue (preview) only works in `inline fun` lambdas — adopt after stabilization
- Multi-dollar strings require `-Xmulti-dollar-interpolation` opt-in in 2.1 (stable in 2.2)

---

## 9. Kotlin 2.1 → 2.2 (June 2025)

### Stable Features (Promoted from Preview)

- **Guard conditions in `when`** — no longer requires compiler flag
- **Non-local `break` and `continue`** in inline functions — now stable
- **Multi-dollar string interpolation** — no longer requires `-Xmulti-dollar-interpolation`

### Context Parameters (Preview)

```kotlin
// 2.2+: implicit context dependencies (preview — requires -Xcontext-parameters)
context(logger: Logger)
fun processOrder(order: Order) {
    logger.info("Processing order: ${order.id}")
}

// Caller provides context
with(myLogger) {
    processOrder(order)
}
```

- Replaces the older experimental "context receivers" feature
- Context parameters are NOT receivers — access members by name, not implicitly
- Enable with `-Xcontext-parameters` compiler option

### Context-Sensitive Resolution (Preview)

```kotlin
// 2.2+: omit type name when expected type is known (preview)
enum class Problem { CONNECTION, TIMEOUT, AUTH }

fun handle(problem: Problem) = when (problem) {
    CONNECTION -> reconnect()  // Instead of Problem.CONNECTION
    TIMEOUT -> retry()
    AUTH -> reauth()
}
```

- Enable with `-Xcontext-sensitive-resolution` compiler option

### Nested Type Aliases (Beta)

```kotlin
// 2.2+: type aliases inside classes
class Graph {
    typealias NodeSet = Set<Node>
    typealias EdgeList = List<Edge>

    fun traverse(visited: NodeSet): EdgeList = ...
}
```

- Enable with `-Xnested-type-aliases` compiler option

### JVM Default Interface Methods

```kotlin
// 2.2+: interface functions compiled to JVM default methods by default
// Configure via compilerOptions:
kotlin {
    compilerOptions {
        jvmDefault = JvmDefaultMode.NO_COMPATIBILITY // No bridge methods
    }
}
```

| Mode               | Behavior                                         |
| ------------------ | ------------------------------------------------ |
| `ENABLE`           | Default methods + compatibility bridges (default)|
| `NO_COMPATIBILITY` | Default methods only (recommended for new code)  |
| `DISABLE`          | Previous behavior (DefaultImpls classes)         |

### `@JvmExposeBoxed` for Value Classes

```kotlin
// 2.2+: expose boxed value class variants to Java
@JvmInline
@JvmExposeBoxed
value class UserId(val value: Long)

// Java can now use: new UserId(42L) — previously inaccessible
```

### Standard Library Stabilizations

- `Base64` API promoted to stable (encode/decode with Default, UrlSafe, Mime schemes)
- `HexFormat` API promoted to stable

### Removals and Deprecations

- Language versions 1.6 and 1.7 no longer supported
- `kotlin-android-extensions` plugin removed — use `kotlin-parcelize` and View Binding
- `kotlinOptions {}` block in Gradle raised to error — use `compilerOptions {}`

### Migration Notes

- Guard conditions, non-local break/continue, multi-dollar strings now stable — safe to adopt
- Replace `kotlinOptions {}` with `compilerOptions {}` in all build files
- Remove `-Xwhen-guards`, `-Xnon-local-break-continue`, `-Xmulti-dollar-interpolation` flags
- Context parameters are still preview in Kotlin 2.3 — evaluate but do not use in production yet
- Nested type aliases and improved `when` exhaustiveness checks are now stable
- If using Java interop with value classes, consider `@JvmExposeBoxed`

---

## 10. Kotlin 2.2 → 2.3 (December 2025)

Kotlin 2.3.0 was released on December 16, 2025.

### Stable Features (Promoted from Beta/Experimental)

The following features have graduated to Stable in Kotlin 2.3.0:

- **Support for nested type aliases** — type aliases can now be declared inside classes
- **Data-flow-based exhaustiveness checks for `when` expressions** — improved compile-time safety

### Features Enabled by Default

Several features are now enabled by default without requiring compiler flags.

### Explicit Backing Fields

```kotlin
// 2.3+: declare explicit backing field with different type
class Counter {
    val count: Int
        field = AtomicInteger(0)  // Backing field is AtomicInteger
        get() = field.get()

    fun increment() {
        count.field.incrementAndGet()  // Direct access to backing field
    }
}
```

### Unused Return Value Checker

- Compiler warns when function return values are not used
- Helps catch bugs where results are accidentally discarded

### Java 25 Support (JVM)

- Kotlin/JVM now supports Java 25 as compilation target
- Bytecode generation for Java 25 features

### Gradle 9.0 Compatibility

- Kotlin Gradle Plugin now compatible with Gradle 9.0
- New API for registering generated sources

### Kotlin/Native Improvements

- Swift export improvements for better interoperability
- Faster release build tasks
- C and Objective-C library import now in Beta

### Kotlin/Wasm Improvements

- Fully qualified names and new exception handling proposal enabled by default
- New compact storage for Latin-1 characters

### Standard Library

- Stable time tracking functionality
- Improved UUID generation and parsing

### Migration Notes

- Review code for unused return values — new warnings may appear
- Update Gradle to 9.0 if using latest Kotlin features
- Java 25 target available for projects requiring latest JVM features

---

## 11. Migration Checklist

### Before Version Upgrade

- [ ] Check deprecated API warnings in current project (`-Werror` flag to treat warnings as errors)
- [ ] Verify Gradle plugin compatibility (kotlin-gradle-plugin, Spring Boot plugin)
- [ ] Verify dependency library Kotlin version compatibility
- [ ] Verify compiler plugin compatibility (kapt, allopen, noarg)

### Upgrade Procedure

1. Change Kotlin version in `gradle.properties`
2. Run clean build (`./gradlew clean build`)
3. Review compile errors and warnings — replace deprecated APIs
4. Run full test suite
5. Validate in CI pipeline

### After Version Upgrade

- [ ] Replace newly deprecated APIs (prepare for future removal)
- [ ] Coordinate new feature adoption with team (code style consistency)
- [ ] Remove explicit `languageVersion`, `apiVersion` settings (use new version defaults)

---

## 12. Deprecated API Replacement Guide

| Deprecated                                | Replacement                        | Since |
| ----------------------------------------- | ---------------------------------- | ----- |
| `inline class`                            | `@JvmInline value class`          | 1.5   |
| `Enum.values()`                           | `Enum.entries`                     | 1.9   |
| `object` (sealed subtype)                 | `data object`                      | 1.9   |
| `kotlinOptions { jvmTarget = "..." }`     | `kotlin { jvmToolchain(N) }`      | 1.8   |
| `@Experimental`                           | `@RequiresOptIn`                   | 1.7   |
| `kotlin-android-extensions` plugin        | View Binding / Compose             | 1.8   |
| `kapt`                                    | KSP                                | 2.0   |
| `kotlin.experimental.ExperimentalTypeInference` | Not needed (builder inference stable) | 1.6 |
| `capitalize()` / `decapitalize()`         | `replaceFirstChar { ... }`         | 1.5   |
| `Iterable.sumBy { }`                      | `Iterable.sumOf { }`              | 1.5   |
| `mapNotNull { }.first()`                  | `firstNotNullOf { }`              | 1.5   |
| `x..y - 1` or `x until y`                | `x..<y`                           | 1.9   |
| `kotlinOptions {}` (Gradle)              | `compilerOptions {}`               | 2.2   |
| `kotlin-android-extensions` plugin       | `kotlin-parcelize` + View Binding  | 2.2   |
| Context receivers (`context(T)`)         | Context parameters (`context(t: T)`) | 2.2 |

---

## 13. Compiler Options Reference

### Recommended Compiler Settings

```kotlin
// build.gradle.kts
kotlin {
    jvmToolchain(21)

    compilerOptions {
        allWarningsAsErrors.set(true)      // Treat warnings as errors
        freeCompilerArgs.addAll(
            "-Xjsr305=strict",             // Strict JSR-305 null annotation handling
            "-opt-in=kotlin.RequiresOptIn"  // Allow opt-in API usage
        )
    }
}
```

### Kotlin 2.2+ Experimental Feature Flags

```kotlin
// build.gradle.kts — opt-in to preview features
kotlin {
    compilerOptions {
        freeCompilerArgs.addAll(
            "-Xcontext-parameters",            // Context parameters (2.2 preview)
            "-Xcontext-sensitive-resolution",   // Context-sensitive resolution (2.2 preview)
            "-Xnested-type-aliases",            // Nested type aliases (2.2 beta)
        )
    }
}
```

### Flexible Compiler Version (2.2+)

```kotlin
// Use a different compiler version than the Gradle plugin version
kotlin {
    compilerVersion.set("2.3.0") // Use newer compiler with older plugin
}
```
