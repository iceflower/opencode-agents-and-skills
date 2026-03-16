---
name: spring-webflux
description: >-
  Spring WebFlux and Kotlin Coroutines conventions.
  Use when writing reactive endpoints, suspend functions in controllers,
  R2DBC repositories, WebClient, Flow pipelines, or SSE streaming.
---

# Spring WebFlux and Coroutines Rules

## 1. WebFlux vs MVC Selection Criteria

### When to Use WebFlux

| Scenario                                    | WebFlux | MVC  |
| ------------------------------------------- | ------- | ---- |
| High concurrency with I/O-bound workloads   | Yes     |      |
| Streaming data (SSE, WebSocket)             | Yes     |      |
| CPU-bound workloads                         |         | Yes  |
| Blocking libraries (JDBC, legacy SDK)       |         | Yes  |
| Team unfamiliar with reactive programming   |         | Yes  |
| Microservice gateway or proxy               | Yes     |      |
| Simple CRUD with moderate traffic           |         | Yes  |

### Key Constraint

- WebFlux runs on a small, fixed thread pool (event loop)
- **Never block the event loop** — blocking calls (JDBC, `Thread.sleep`, synchronized) will starve all requests
- If you must use blocking libraries, use `Dispatchers.IO` or `Schedulers.boundedElastic()`

---

## 2. Kotlin Coroutines with Spring WebFlux

### Controller Layer

```kotlin
@RestController
@RequestMapping("/api/v1/users")
class UserController(
    private val userService: UserService
) {
    // suspend fun for single result
    @GetMapping("/{id}")
    suspend fun getUser(@PathVariable id: Long): UserResponse =
        userService.findById(id)

    // Flow for streaming results
    @GetMapping
    fun listUsers(): Flow<UserResponse> =
        userService.findAll()

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    suspend fun createUser(@Valid @RequestBody request: CreateUserRequest): UserResponse =
        userService.create(request)
}
```

### Service Layer

```kotlin
@Service
class UserService(
    private val userRepository: UserRepository,
    private val notificationClient: NotificationClient
) {
    suspend fun findById(id: Long): UserResponse =
        userRepository.findById(id)
            ?.toResponse()
            ?: throw EntityNotFoundException("User", id)

    fun findAll(): Flow<UserResponse> =
        userRepository.findAll().map { it.toResponse() }

    suspend fun create(request: CreateUserRequest): UserResponse {
        val user = userRepository.save(request.toEntity())
        notificationClient.sendWelcome(user.id)
        return user.toResponse()
    }
}
```

### Repository Layer (R2DBC)

```kotlin
interface UserRepository : CoroutineCrudRepository<User, Long> {
    suspend fun findByEmail(email: String): User?
    fun findByStatus(status: UserStatus): Flow<User>

    @Query("SELECT * FROM users WHERE name LIKE :keyword%")
    fun searchByNamePrefix(keyword: String): Flow<User>
}
```

### Coroutine Rules

- Controller: `suspend fun` for single results, `Flow<T>` return for streaming
- Service: `suspend fun` for business logic, `Flow<T>` for collection operations
- Repository: Use `CoroutineCrudRepository` (not `ReactiveCrudRepository`)
- Never use `runBlocking` in request-handling code — it blocks the event loop
- Use `coroutineScope` for structured parallel operations

---

## 3. Parallel Execution

### Structured Concurrency with coroutineScope

```kotlin
suspend fun getDashboard(userId: Long): DashboardResponse = coroutineScope {
    val profile = async { userService.findById(userId) }
    val orders = async { orderService.findRecentByUserId(userId) }
    val notifications = async { notificationService.findUnread(userId) }

    DashboardResponse(
        profile = profile.await(),
        orders = orders.await(),
        notifications = notifications.await()
    )
}
```

### Rules

- Always use `coroutineScope` or `supervisorScope` — never `GlobalScope`
- Use `async` + `await` for parallel I/O operations
- Use `supervisorScope` when child failures should not cancel siblings
- `coroutineScope` cancels all children if any child fails — use for all-or-nothing operations

---

## 4. Flow Patterns

### Flow Operators

```kotlin
// Transform
fun findActiveUsers(): Flow<UserResponse> =
    userRepository.findAll()
        .filter { it.status == UserStatus.ACTIVE }
        .map { it.toResponse() }

// Batch processing (collect to list, then chunk)
suspend fun batchProcess(items: Flow<Item>) {
    items.toList()
        .chunked(100)
        .forEach { batch ->
            processBatch(batch)
        }
}

// Error handling in Flow
fun fetchData(): Flow<Data> =
    dataRepository.findAll()
        .catch { e ->
            log.error("Failed to fetch data", e)
            emit(Data.fallback())
        }
        .onCompletion { cause ->
            if (cause != null) log.warn("Flow completed with error", cause)
        }
```

### Flow vs List

| Use Case                         | Recommended |
| -------------------------------- | ----------- |
| Large result set (1000+ items)   | `Flow`      |
| Need all items before processing | `List`      |
| Streaming to client (SSE)        | `Flow`      |
| Small, bounded collection        | `List`      |
| Intermediate pipeline operations | `Flow`      |

### Flow Rules

- Prefer `Flow` over `List` for database queries that may return large result sets
- `Flow` is cold — collection does not start until `collect` is called
- Never collect a `Flow` inside another `Flow.map` — use `flatMapConcat` or `flatMapMerge`
- Use `flowOn(Dispatchers.IO)` to shift blocking operations off the event loop

---

## 5. WebClient (HTTP Client)

### Configuration

```kotlin
@Configuration
class WebClientConfig(
    private val props: HttpClientProperties
) {
    @Bean
    fun externalApiWebClient(): WebClient =
        WebClient.builder()
            .baseUrl(props.baseUrl)
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .clientConnector(
                ReactorClientHttpConnector(
                    HttpClient.create()
                        .responseTimeout(props.readTimeout)
                        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, props.connectTimeout.toMillis().toInt())
                )
            )
            .build()
}
```

### Coroutine-Style Usage

```kotlin
@Component
class PaymentClient(
    private val webClient: WebClient
) {
    suspend fun getPayment(paymentId: String): PaymentResponse =
        webClient.get()
            .uri("/payments/{id}", paymentId)
            .retrieve()
            .awaitBody()

    suspend fun createPayment(request: PaymentRequest): PaymentResponse =
        webClient.post()
            .uri("/payments")
            .bodyValue(request)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError) { response ->
                Mono.error(PaymentValidationException("Payment validation failed"))
            }
            .awaitBody()
}
```

### WebClient Rules

- Use `awaitBody()`, `awaitExchange()` for coroutine integration
- Never use `block()` — it defeats the purpose of non-blocking
- Configure timeouts explicitly — WebClient defaults may be infinite
- Use `onStatus` for HTTP error mapping before `awaitBody()`

---

## 6. Error Handling

### Coroutine Exception Handling

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException::class)
    suspend fun handleBusinessException(e: BusinessException): ResponseEntity<ErrorResponse> {
        log.warn("Business exception: ${e.errorCode.code}", e)
        return ResponseEntity
            .status(e.errorCode.status)
            .body(ErrorResponse.of(e.errorCode, e.message))
    }

    @ExceptionHandler(WebClientResponseException::class)
    suspend fun handleWebClientException(e: WebClientResponseException): ResponseEntity<ErrorResponse> {
        log.error("External API error: ${e.statusCode}", e)
        return ResponseEntity
            .status(HttpStatus.BAD_GATEWAY)
            .body(ErrorResponse.of(ErrorCode.EXTERNAL_API_ERROR))
    }
}
```

### Timeout Handling

```kotlin
suspend fun fetchWithTimeout(): Data =
    withTimeout(5000L) {
        externalClient.fetchData()
    }

// Or catch timeout
suspend fun fetchSafely(): Data? =
    try {
        withTimeout(5000L) { externalClient.fetchData() }
    } catch (e: TimeoutCancellationException) {
        log.warn("External API timeout", e)
        null
    }
```

### Error Handling Rules

- `@RestControllerAdvice` works the same way as in MVC
- Use `withTimeout` for coroutine-level timeout control
- `CancellationException` should not be caught — it is the cancellation signal for structured concurrency
- Wrap external API errors in domain exceptions at the client boundary

---

## 7. R2DBC Database Access

### Dependencies

```kotlin
// build.gradle.kts
implementation("org.springframework.boot:spring-boot-starter-data-r2dbc")
runtimeOnly("org.postgresql:r2dbc-postgresql")
// Or: runtimeOnly("io.asyncer:r2dbc-mysql")
```

### Configuration

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    pool:
      initial-size: ${R2DBC_POOL_INITIAL:5}
      max-size: ${R2DBC_POOL_MAX:20}
      max-idle-time: ${R2DBC_POOL_MAX_IDLE:30m}
```

### Entity Mapping

```kotlin
@Table("users")
data class User(
    @Id
    val id: Long? = null,
    val name: String,
    val email: String,
    val status: UserStatus = UserStatus.ACTIVE,
    val createdAt: Instant = Instant.now(),
    val updatedAt: Instant = Instant.now()
)
```

### R2DBC Rules

- R2DBC does not support lazy loading or JPA relationships — fetch joins manually
- Use `@Query` with SQL for join queries — no JPQL support
- Use `DatabaseClient` for complex queries that `CoroutineCrudRepository` cannot express
- Schema migration still uses Flyway/Liquibase (they run blocking at startup, which is acceptable)

---

## 8. Server-Sent Events (SSE)

```kotlin
@GetMapping("/stream", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
fun streamEvents(): Flow<ServerSentEvent<EventData>> =
    eventService.subscribe()
        .map { event ->
            ServerSentEvent.builder(event)
                .id(event.id)
                .event(event.type)
                .build()
        }
```

---

## 9. Testing

### Controller Test

```kotlin
@WebFluxTest(UserController::class)
class UserControllerTest(
    @Autowired private val webTestClient: WebTestClient,
    @MockkBean private val userService: UserService
) {
    @Test
    fun `should return user by id`() {
        coEvery { userService.findById(1L) } returns userResponse

        webTestClient.get()
            .uri("/api/v1/users/1")
            .exchange()
            .expectStatus().isOk
            .expectBody<UserResponse>()
            .isEqualTo(userResponse)
    }
}
```

### Coroutine Test

```kotlin
@Test
fun `should fetch dashboard in parallel`() = runTest {
    coEvery { userService.findById(1L) } returns profile
    coEvery { orderService.findRecentByUserId(1L) } returns orders

    val result = dashboardService.getDashboard(1L)

    assertThat(result.profile).isEqualTo(profile)
    assertThat(result.orders).isEqualTo(orders)
}
```

### Testing Rules

- Use `runTest` from `kotlinx-coroutines-test` for suspend function tests
- Use `WebTestClient` for controller integration tests
- Use `coEvery` / `coVerify` from MockK for coroutine mock interactions
- Use `Turbine` library for `Flow` assertion

---

## 10. Blocking Code Integration

### When Blocking is Unavoidable

```kotlin
// Shift blocking call to IO dispatcher
suspend fun legacyOperation(): Result =
    withContext(Dispatchers.IO) {
        legacyBlockingClient.call()
    }

// In Flow pipeline — flowOn affects operators ABOVE it
fun processWithBlocking(): Flow<Data> =
    sourceFlow
        .map { blockingTransform(it) }
        .flowOn(Dispatchers.IO)
```

### Rules

- Use `Dispatchers.IO` for JDBC, file I/O, or legacy blocking SDK calls
- Never call blocking code on `Dispatchers.Default` or the event loop
- If most of the application is blocking, use Spring MVC instead of WebFlux
- `withContext(Dispatchers.IO)` is acceptable for isolated blocking calls, not as a general pattern

---

## 11. Anti-Patterns

- Using `runBlocking` in request handlers (blocks event loop threads)
- Using `block()` on Mono/Flux (defeats non-blocking purpose)
- Mixing Reactor API (`Mono`, `Flux`) with Coroutines without bridge functions (`awaitSingle`, `asFlow`)
- Using `GlobalScope.launch` (unstructured concurrency, memory leaks)
- Catching `CancellationException` without rethrowing (breaks structured concurrency)
- Using JDBC/JPA with WebFlux (blocking drivers on non-blocking runtime)
- Creating unbounded `Flow` without backpressure consideration
- Using `Thread.sleep()` instead of `delay()` in coroutine context
