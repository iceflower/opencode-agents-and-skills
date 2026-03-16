---
name: java-kotlin-interop
description: >-
  Java-Kotlin interoperability guide covering platform types, null safety with JSpecify,
  JVM annotations (@JvmStatic, @JvmOverloads, @JvmExposeBoxed, @Throws), collection interop,
  coroutine-Java bridging, SAM conversion, and Spring-specific patterns for mixed projects.
---

# Java-Kotlin Interoperability Rules

## 1. Platform Types and Null Safety

### The Core Problem

Java types without nullability annotations are treated as **platform types** (`T!`) in Kotlin — neither nullable nor non-nullable. This bypasses Kotlin's null-safety guarantees.

```kotlin
// Java method: String getName() — no annotation
val name = javaObject.name  // Type: String! (platform type)
name.length                  // Compiles, but may throw NPE at runtime
```

### Solution: Nullability Annotations

```java
// Java — annotate all public API boundaries
import org.jspecify.annotations.NullMarked;
import org.jspecify.annotations.Nullable;

@NullMarked  // All types non-null by default in this class
public class UserService {
    public User findById(long id) { ... }              // Non-null return
    public @Nullable User findByEmail(String email) { ... }  // Nullable return
}
```

```kotlin
// Kotlin — now properly typed
val user: User = userService.findById(1L)          // Non-null
val maybeUser: User? = userService.findByEmail(email)  // Nullable
```

### Annotation Priority (Kotlin Recognition Order)

| Priority | Annotation Source        | Package                              |
| -------- | ------------------------ | ------------------------------------ |
| 1        | JSpecify                 | `org.jspecify.annotations`           |
| 2        | JetBrains               | `org.jetbrains.annotations`          |
| 3        | Android                  | `androidx.annotation`                |
| 4        | JSR-305                  | `javax.annotation`                   |
| 5        | FindBugs                 | `edu.umd.cs.findbugs.annotations`    |
| 6        | Eclipse                  | `org.eclipse.jdt.annotation`         |
| 7        | Lombok                   | `lombok`                             |

### Spring Framework 7 / Spring Boot 4: JSpecify Integration

Spring Framework 7+ uses JSpecify annotations throughout the entire codebase with `@NullMarked` on all packages.

```kotlin
// Before (Spring 6 / Boot 3): platform types everywhere
val user = userRepository.findById(id)  // User! — platform type
user.name                                // May NPE

// After (Spring 7 / Boot 4 + Kotlin 2.1+): proper null safety
val user = userRepository.findById(id)  // User — non-null
val maybe = userRepository.findByEmail(email)  // User? — nullable
maybe?.name                              // Compiler-enforced safety
```

- Kotlin 2.2+ automatically translates JSpecify annotations to Kotlin nullability
- Platform types (`T!`) are eliminated for Spring APIs
- Generic types and arrays also carry nullability: `List<String>` not `List<String!>!`

### Rules

- **Java side**: Always annotate public APIs with `@NullMarked` (class/package level) and `@Nullable` (specific fields/returns)
- **Kotlin side**: Never use `!!` on platform types — assign to explicitly typed variable first
- **Mixed project**: Use JSpecify over JSR-305 for new code (better Kotlin integration)
- **Spring Boot 4**: No extra work needed — Spring APIs are already fully annotated

---

## 2. Calling Kotlin from Java

### Companion Object Members

```kotlin
class UserService {
    companion object {
        @JvmStatic
        fun defaultInstance(): UserService = UserService()

        @JvmField
        val MAX_USERS = 1000

        const val VERSION = "1.0"  // Inlined at compile time
    }
}
```

```java
// Java — clean access with @JvmStatic and @JvmField
UserService service = UserService.defaultInstance();  // Static call
int max = UserService.MAX_USERS;                      // Direct field
String version = UserService.VERSION;                 // Constant

// Without annotations: requires .Companion
UserService service = UserService.Companion.defaultInstance();
```

### Default Parameters

```kotlin
class Circle @JvmOverloads constructor(
    val centerX: Int,
    val centerY: Int,
    val radius: Double = 1.0
) {
    @JvmOverloads
    fun draw(color: String = "black", filled: Boolean = true) { ... }
}
```

```java
// Java — overloaded constructors and methods generated
new Circle(10, 20);         // radius defaults to 1.0
new Circle(10, 20, 5.0);   // explicit radius
circle.draw();              // both defaults
circle.draw("red");         // filled defaults to true
circle.draw("red", false);  // all explicit
```

### Checked Exceptions

```kotlin
// Kotlin — no checked exceptions by default
@Throws(IOException::class)  // Required for Java interop
fun readFile(path: String): String {
    return File(path).readText()
}
```

```java
// Java — can now catch the declared exception
try {
    String content = FileUtilKt.readFile("data.txt");
} catch (IOException e) {
    // Handle exception
}
```

### Value Classes (Kotlin 2.2+)

```kotlin
@JvmInline
@JvmExposeBoxed  // Expose boxed constructors and methods to Java
value class UserId(val value: Long)

@JvmInline
@JvmExposeBoxed
value class Email(val value: String)

// Function using value classes
@JvmExposeBoxed
fun findUser(id: UserId): User? = ...
```

```java
// Java — boxed variants available
UserId id = new UserId(42L);           // Constructor accessible
User user = UserServiceKt.findUser(id); // Boxed parameter accepted
```

- Without `@JvmExposeBoxed`: value classes are unboxed (mangled names, unusable from Java)
- Module-wide option: compile with `-Xjvm-expose-boxed`

### Package-Level Functions

```kotlin
// FileUtils.kt
@file:JvmName("FileUtils")  // Custom class name for Java
package com.example.util

fun readContent(path: String): String = ...
```

```java
// Java
String content = FileUtils.readContent("data.txt");
// Without @JvmName: FileUtilsKt.readContent(...)
```

### Property Access

```kotlin
class User(
    val name: String,           // Java: getName()
    var email: String,          // Java: getEmail(), setEmail()
    @get:JvmName("isVerified")
    val verified: Boolean       // Java: isVerified()
)
```

### Annotation Summary

| Annotation         | Purpose                                         | When to Use                     |
| ------------------ | ----------------------------------------------- | ------------------------------- |
| `@JvmStatic`       | Generate static method                          | Companion object functions      |
| `@JvmField`        | Expose as public field (no getter/setter)        | Companion object properties     |
| `@JvmOverloads`    | Generate overloads for default parameters        | Functions called from Java      |
| `@JvmName`         | Specify JVM method/class name                    | Name clashes, file-level funcs  |
| `@Throws`          | Declare checked exceptions                       | Functions throwing exceptions   |
| `@JvmExposeBoxed`  | Expose boxed value class for Java                | Value classes used from Java    |
| `@JvmWildcard`     | Force wildcard in generated Java signature       | Generic variance control        |
| `@JvmSuppressWildcards` | Suppress wildcard in generated Java signature | Generic variance control   |

---

## 3. Collection Interop

### Read-Only vs Mutable Mapping

| Java Type        | Kotlin Read-Only       | Kotlin Mutable             |
| ---------------- | ---------------------- | -------------------------- |
| `java.util.List` | `kotlin.List`          | `kotlin.MutableList`       |
| `java.util.Set`  | `kotlin.Set`           | `kotlin.MutableSet`        |
| `java.util.Map`  | `kotlin.Map`           | `kotlin.MutableMap`        |
| `java.util.Collection` | `kotlin.Collection` | `kotlin.MutableCollection` |

### Common Pitfalls

```kotlin
// Pitfall 1: Java can mutate Kotlin's read-only collection
fun getNames(): List<String> = listOf("Alice", "Bob")

// Java code can cast and mutate:
// List<String> names = getNames();
// ((java.util.ArrayList<String>) names).add("Charlie"); // UnsupportedOperationException

// Pitfall 2: Platform type collections — mutability unknown
fun processJavaList(list: MutableList<String>) {
    list.add("item")  // OK — explicitly mutable
}
fun processJavaList(list: List<String>) {
    // Cannot add — Kotlin treats as read-only
}
```

### Rules

- Return `List` (read-only) from Kotlin APIs — Java consumers cannot mutate accidentally
- Accept `MutableList` in Kotlin parameters when Java caller needs to mutate
- Defensive copy when receiving collections from Java: `list.toList()` or `list.toMutableList()`
- Use `List.copyOf()` in Java when passing to Kotlin to ensure immutability

---

## 4. Generics and Variance

### Declaration-Site Variance → Use-Site Wildcards

```kotlin
// Kotlin: declaration-site variance
class Box<out T>(val value: T)  // Covariant
class Consumer<in T> { fun consume(item: T) {} }  // Contravariant

// Generated Java:
// Box<? extends T>   for out
// Consumer<? super T> for in
```

### Controlling Wildcard Generation

```kotlin
// Force wildcard
fun boxDerived(value: Derived): Box<@JvmWildcard Derived> = Box(value)
// Java: Box<? extends Derived>

// Suppress wildcard
fun unboxBase(box: Box<@JvmSuppressWildcards Base>): Base = box.value
// Java: Box<Base> (no wildcard)
```

### Reified Type Parameters

```kotlin
// Kotlin — inline + reified preserves type at runtime
inline fun <reified T> parseJson(json: String): T {
    return objectMapper.readValue(json, T::class.java)
}

// Cannot be called from Java — reified is a Kotlin-only feature
// Java alternative: pass Class<T> explicitly
fun <T> parseJson(json: String, type: Class<T>): T {
    return objectMapper.readValue(json, type)
}
```

### Rules

- Provide non-reified overload with `Class<T>` parameter for Java consumers
- Use `@JvmWildcard` / `@JvmSuppressWildcards` to control Java signature when needed
- Java raw types become `Any!` in Kotlin — avoid raw types in interop boundaries

---

## 5. Coroutines and Java Interop

### Exposing Suspend Functions to Java

```kotlin
// Kotlin suspend function
suspend fun fetchUser(id: Long): User { ... }

// Java cannot call suspend functions directly
// Solution 1: Provide a CompletableFuture wrapper
fun fetchUserAsync(id: Long): CompletableFuture<User> =
    CoroutineScope(Dispatchers.IO).future { fetchUser(id) }

// Solution 2: Provide a blocking wrapper (for simple cases)
@JvmStatic
fun fetchUserBlocking(id: Long): User = runBlocking { fetchUser(id) }
```

### Flow to Java

```kotlin
// Kotlin Flow
fun userStream(): Flow<User> = ...

// Java-friendly wrapper using Reactor or RxJava
fun userFlux(): Flux<User> = userStream().asFlux()

// Or using Publisher
fun userPublisher(): Publisher<User> = userStream().asPublisher()
```

### Rules

- Never expose raw `suspend fun` as public API consumed by Java — wrap in `CompletableFuture`
- Use `kotlinx-coroutines-jdk8` for `future {}` builder
- Use `kotlinx-coroutines-reactor` for `Flow.asFlux()` / `Flow.asPublisher()` conversion
- `runBlocking` wrappers are acceptable for CLI tools but NOT for server request handlers

---

## 6. SAM Conversion

### Java SAM Interface in Kotlin

```kotlin
// Java interface
// public interface Predicate<T> { boolean test(T t); }

// Kotlin — automatic SAM conversion
val isAdult = Predicate<User> { it.age >= 18 }
users.stream().filter { it.age >= 18 }
```

### Kotlin Fun Interface in Java

```kotlin
// Kotlin — fun interface enables SAM conversion
fun interface Validator<T> {
    fun validate(value: T): Boolean
}
```

```java
// Java — lambda works with fun interface
Validator<String> notEmpty = s -> !s.isEmpty();
```

### Rules

- Use `fun interface` in Kotlin when Java consumers should use lambdas
- Regular Kotlin interfaces do NOT support SAM conversion from Java — must use `fun interface`
- If a Kotlin interface has multiple abstract methods, Java must use anonymous class

---

## 7. Keyword and Name Conflicts

### Kotlin Keywords as Java Identifiers

```kotlin
// Java method named with Kotlin keyword — use backticks
javaObject.`is`(value)
javaObject.`when`
javaObject.`object`
javaObject.`in`(collection)
```

### `internal` Visibility from Java

```kotlin
internal fun processInternal() { ... }

// Java can access (public in bytecode) but name is mangled:
// processInternal$module_name()
// This is intentional — discourages accidental use from Java
```

### Rules

- Avoid using Kotlin keywords (`is`, `when`, `object`, `in`, `fun`, `val`, `var`) as Java identifiers in interop boundaries
- `internal` Kotlin members are accessible from Java but name-mangled — do not depend on them from Java
- Use `@JvmName` to provide clean Java-friendly names when needed

---

## 8. Spring-Specific Interop Patterns

### Kotlin Extensions Used from Java

```kotlin
// Kotlin extension function
fun User.toResponse(): UserResponse = UserResponse(id, name, email)

// Java — called as static method on the generated Kt class
UserResponse response = UserMappingsKt.toResponse(user);
```

- Extension functions compile to static methods with receiver as first parameter
- Use `@file:JvmName("UserMappings")` for cleaner Java access

### Spring Configuration in Mixed Projects

```kotlin
// Kotlin @Configuration is open by default (allopen plugin)
@Configuration
class AppConfig {
    @Bean
    fun userService(repo: UserRepository): UserService = UserService(repo)
}

// Java @Configuration must use CGLIB proxying
@Configuration
public class JavaConfig {
    @Bean
    public PaymentService paymentService() { return new PaymentService(); }
}
```

- Kotlin's `allopen` plugin makes `@Configuration`, `@Service`, etc. automatically open
- Java classes must be non-final for CGLIB proxying (or use `proxyBeanMethods = false`)

### JPA Entities in Mixed Projects

```kotlin
// Kotlin entity — requires allopen + noarg plugins
@Entity
class User(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    var name: String,
    var email: String
)
```

```java
// Java entity — works without plugins
@Entity
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    // JPA requires no-arg constructor — Java generates implicitly with default visibility
}
```

- Kotlin JPA entities need `kotlin-jpa` (noarg) plugin for no-arg constructor generation
- Kotlin JPA entities need `kotlin-allopen` plugin to make classes non-final

---

## 9. Build Configuration for Mixed Projects

### Gradle (Kotlin DSL)

```kotlin
// build.gradle.kts
plugins {
    java
    kotlin("jvm") version "2.3.0"
    kotlin("plugin.spring") version "2.3.0"   // allopen for Spring
    kotlin("plugin.jpa") version "2.3.0"       // noarg for JPA
}

// Source directories
sourceSets {
    main {
        java.srcDirs("src/main/java")
        kotlin.srcDirs("src/main/kotlin")
    }
}

// Ensure Java and Kotlin compile together
tasks.withType<JavaCompile> {
    sourceCompatibility = "21"
    targetCompatibility = "21"
}

kotlin {
    jvmToolchain(21)
    compilerOptions {
        freeCompilerArgs.addAll("-Xjsr305=strict")
    }
}
```

### Compilation Order

- Kotlin compiler runs first — can see Java sources
- Java compiler runs second — can see Kotlin compiled classes
- **Circular dependencies between Java and Kotlin files are supported** (both compilers handle cross-references)

---

## 10. Migration Strategy

### Gradual Java → Kotlin Migration

| Step | Action                                  | Risk Level |
| ---- | --------------------------------------- | ---------- |
| 1    | Add Kotlin plugin to build              | Low        |
| 2    | Write new test code in Kotlin           | Low        |
| 3    | Write new utility/extension classes     | Low        |
| 4    | Convert data classes (DTOs, responses)  | Low        |
| 5    | Convert service layer classes           | Medium     |
| 6    | Convert controller layer                | Medium     |
| 7    | Convert entity classes (requires plugins)| Medium    |

### Migration Rules

- Convert bottom-up (least-depended-on classes first)
- Add nullability annotations to Java code before converting callers to Kotlin
- Use IntelliJ's "Convert Java File to Kotlin" as starting point, then refine
- Keep test coverage high — run tests after each conversion
- Do not convert and refactor simultaneously — convert first, then refactor

---

## 11. Anti-Patterns

- Using `!!` on platform types — assign to typed variable or add Java annotation instead
- Exposing `suspend fun` directly to Java consumers — wrap in `CompletableFuture`
- Exposing `Flow` to Java — wrap in `Flux` or `Publisher`
- Using regular Kotlin interfaces for Java SAM consumption — use `fun interface`
- Relying on Kotlin `internal` visibility for encapsulation from Java — it is `public` in bytecode
- Mixing `javax` and `jakarta` annotations in the same project
- Not adding `@JvmStatic` / `@JvmOverloads` / `@Throws` on Kotlin code called from Java
- Using `data class` for JPA entities without understanding `equals`/`hashCode` implications
- Not using `@JvmExposeBoxed` for value classes consumed from Java
- Ignoring platform types — always determine proper nullability at interop boundaries
