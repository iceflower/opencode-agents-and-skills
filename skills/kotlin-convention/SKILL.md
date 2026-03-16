---
name: kotlin-convention
description: Kotlin coding conventions and idioms. Use when writing or reviewing
  Kotlin code.
---

# Kotlin Convention Rules

## 1. Null Safety

### Preferred Patterns

```kotlin
// Use safe call + Elvis for default values
val name = user?.name ?: "Unknown"

// Use requireNotNull for preconditions (throws IllegalArgumentException)
val userId = requireNotNull(request.userId) { "userId must not be null" }

// Use checkNotNull for state validation (throws IllegalStateException)
val session = checkNotNull(currentSession) { "Session not initialized" }

// Use let for null-conditional execution
user?.let { sendNotification(it) }
```

### Null Anti-Patterns

```kotlin
// Never use !! in production code
val name = user!!.name  // Bad: crashes with NPE

// Avoid unnecessary null types
fun getUser(): User? // Bad if it never returns null
fun getUser(): User  // Good: non-null return when guaranteed
```

---

## 2. Data Class vs Class

| Use | When |
| --- | --- |
| `data class` | DTOs, request/response objects, value objects, config properties |
| `class` | Services, repositories, entities with mutable state, classes with complex behavior |
| `value class` | Single-field wrappers for type safety (IDs, amounts) |
| `sealed class` | Restricted type hierarchies (states, results, error types) |
| `object` | Singletons, utility collections, companion factories |

```kotlin
// Value class for type-safe IDs
@JvmInline
value class UserId(val value: Long)

// Sealed class for result types
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Failure(val error: ErrorCode) : Result<Nothing>()
}
```

---

## 3. Extension Functions

### Good Use Cases

```kotlin
// Converting between layers (Entity ↔ DTO)
fun User.toResponse() = UserResponse(id = id, name = name, email = email)

// Adding domain-specific utility
fun String.toSlug() = lowercase().replace(Regex("[^a-z0-9]+"), "-").trim('-')

// Scoping to specific contexts
fun RestClient.ResponseSpec.orThrow(message: String): RestClient.ResponseSpec =
    onStatus(HttpStatusCode::isError) { _, _ -> throw ExternalApiException(message) }
```

### Extension Anti-Patterns

- Extension functions that access private/internal state of the receiver
- Extension functions that modify mutable state
- Overusing extensions where a regular method on the class is more appropriate

---

## 4. Collection Operations

### Prefer Kotlin Standard Library

```kotlin
// Transforming
val names = users.map { it.name }
val activeUsers = users.filter { it.isActive }
val userMap = users.associateBy { it.id }

// Aggregating
val totalAmount = orders.sumOf { it.amount }
val grouped = orders.groupBy { it.status }

// Null-safe collections
val firstActive = users.firstOrNull { it.isActive }
val nicknames = users.mapNotNull { it.nickname }
```

### Performance Considerations

- Use `asSequence()` for chains of 3+ operations on large collections
- Use `buildList` / `buildMap` for constructing collections conditionally
- Avoid `flatMap` inside loops — prefer restructuring

---

## 5. Coroutines (for async APIs)

### Structured Concurrency

```kotlin
// Use coroutineScope for parallel operations
suspend fun fetchDashboard(userId: Long): Dashboard = coroutineScope {
    val profile = async { userService.getProfile(userId) }
    val orders = async { orderService.getRecentOrders(userId) }
    Dashboard(profile.await(), orders.await())
}
```

### Spring Integration

- Use `suspend fun` in controllers for WebFlux/reactive endpoints
- Use `@Async` with `CompletableFuture` for MVC-based projects
- Never use `GlobalScope` — always use structured concurrency
- Use `Dispatchers.IO` for blocking I/O operations

---

## 6. Naming Conventions

| Element | Convention | Example |
| --- | --- | --- |
| Class | PascalCase | `UserService`, `OrderResponse` |
| Function | camelCase | `findByEmail`, `calculateTotal` |
| Property | camelCase | `userName`, `isActive` |
| Constant | SCREAMING_SNAKE | `MAX_RETRY_COUNT`, `DEFAULT_PAGE_SIZE` |
| Package | lowercase dot-separated | `com.example.app` |
| Enum value | SCREAMING_SNAKE | `PENDING`, `IN_PROGRESS` |

---

## 7. Spring Boot Specific

### Constructor Injection (preferred)

```kotlin
@Service
class UserService(
    private val userRepository: UserRepository,
    private val eventPublisher: ApplicationEventPublisher
) {
    // No @Autowired needed — single constructor auto-wired
}
```

### Configuration Properties

```kotlin
@ConfigurationProperties(prefix = "app.feature")
data class FeatureProperties(
    val enabled: Boolean = false,
    val maxRetries: Int = 3,
    val timeout: Duration = Duration.ofSeconds(30)
)
```

### Spring Boot Anti-Patterns

- `@Autowired` field injection — use constructor injection
- `lateinit var` for injected dependencies — use constructor parameters
- `open` classes/methods for Spring proxying — use `allopen` plugin instead
