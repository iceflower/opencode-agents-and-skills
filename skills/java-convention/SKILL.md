---
name: java-convention
description: >-
  Java coding conventions and modern idioms with Spring Boot integration.
  Use when writing or reviewing Java code, including records, sealed classes,
  pattern matching, virtual threads, and Spring Boot patterns.
---

# Java Convention Rules

## 1. Null Handling

### Preferred Patterns

```java
// Use Optional for return types that may be absent
public Optional<User> findByEmail(String email) {
    return Optional.ofNullable(userRepository.findByEmail(email));
}

// Chain Optional operations
String displayName = findByEmail(email)
    .map(User::getName)
    .orElse("Unknown");

// Use Objects.requireNonNull for preconditions
public void process(Order order) {
    Objects.requireNonNull(order, "order must not be null");
    Objects.requireNonNull(order.getUserId(), "order.userId must not be null");
}
```

### Null Anti-Patterns

```java
// Never return null from a collection method — return empty collection
public List<Order> findOrders(Long userId) {
    List<Order> orders = repository.findByUserId(userId);
    return orders != null ? orders : List.of(); // Bad: should never be null
}

// Never use Optional as a field or parameter type
private Optional<String> name; // Bad: use @Nullable or default value
public void setName(Optional<String> name) {} // Bad: just use @Nullable

// Never call Optional.get() without isPresent check — use orElseThrow
user.get(); // Bad: may throw NoSuchElementException
user.orElseThrow(() -> new EntityNotFoundException("User", id)); // Good
```

### @Nullable and @NonNull Annotations

```java
// Use JSR-305 or JSpecify annotations at API boundaries
public @NonNull List<User> findAll() { ... }
public @Nullable User findByEmail(@NonNull String email) { ... }
```

### Null Handling Rules

| Scenario                        | Recommended Approach                 |
| ------------------------------- | ------------------------------------ |
| Method return (may be absent)   | `Optional<T>`                        |
| Collection return               | Empty collection, never null         |
| Method parameter                | `@Nullable` annotation or overload   |
| Field                           | Default value or `@Nullable`         |
| External API response           | Null-check at boundary               |

---

## 2. Records, Sealed Classes, and Enums

### Records (Java 16+)

```java
// DTOs, request/response objects, value objects
public record UserResponse(
    Long id,
    String name,
    String email,
    UserStatus status
) {
    // Compact constructor for validation
    public UserResponse {
        Objects.requireNonNull(name, "name must not be null");
        Objects.requireNonNull(email, "email must not be null");
    }

    // Static factory method
    public static UserResponse from(User user) {
        return new UserResponse(user.getId(), user.getName(),
            user.getEmail(), user.getStatus());
    }
}
```

### Sealed Classes (Java 17+)

```java
public sealed interface Result<T> permits Success, Failure {
    record Success<T>(T data) implements Result<T> {}
    record Failure<T>(ErrorCode errorCode, String message) implements Result<T> {}
}
```

### Type Selection Criteria

| Use                  | When                                                    |
| -------------------- | ------------------------------------------------------- |
| `record`             | DTOs, request/response, value objects, immutable data   |
| `class`              | Services, entities with mutable state, complex behavior |
| `sealed interface`   | Restricted type hierarchies (results, states, commands) |
| `enum`               | Fixed set of constants with behavior                    |

### Record Rules

- Records are immutable and final — suitable for DTOs and value objects
- Use compact constructors for validation, not custom constructors
- Records auto-generate `equals()`, `hashCode()`, `toString()` — do not override unless necessary
- Records cannot extend classes (but can implement interfaces)

---

## 3. Pattern Matching

### Pattern Matching for instanceof (Java 16+)

```java
// Before
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// After: pattern variable
if (obj instanceof String s) {
    System.out.println(s.length());
}

// With negation
if (!(obj instanceof String s)) {
    return;
}
// s is in scope here
```

### Pattern Matching for Switch (Java 21+)

```java
// Type pattern matching
public String describe(Object obj) {
    return switch (obj) {
        case Integer i when i > 0 -> "positive: " + i;
        case Integer i            -> "non-positive: " + i;
        case String s             -> "string: " + s;
        case null                 -> "null";
        default                   -> "other: " + obj;
    };
}

// Sealed class exhaustive matching
public String handle(Result<?> result) {
    return switch (result) {
        case Result.Success<?> s -> "Success: " + s.data();
        case Result.Failure<?> f -> "Error: " + f.message();
        // No default needed — sealed type is exhaustive
    };
}
```

### Record Patterns (Java 21+)

```java
// Destructure records in pattern matching
public String format(Object obj) {
    return switch (obj) {
        case UserResponse(var id, var name, var email, var status) ->
            name + " (#" + id + ")";
        default -> obj.toString();
    };
}

// Nested record patterns
record Address(String city, String street) {}
record Person(String name, Address address) {}

if (obj instanceof Person(var name, Address(var city, var street))) {
    System.out.println(name + " lives in " + city);
}
```

### Unnamed Patterns (Java 22+)

```java
// Use _ for unused pattern variables (Java 22+)
if (obj instanceof Person(var name, Address(var city, _))) {
    System.out.println(name + " lives in " + city);
}
```

---

## 4. Text Blocks and String Templates

### Text Blocks (Java 15+)

```java
// Multi-line strings
String json = """
    {
        "name": "%s",
        "email": "%s"
    }
    """.formatted(name, email);

// SQL queries
String query = """
    SELECT u.id, u.name, u.email
    FROM users u
    WHERE u.status = 'ACTIVE'
      AND u.created_at > :cutoff
    ORDER BY u.name
    """;
```

### String Formatting

```java
// Prefer formatted() over String.format() for readability
String message = "User %s (ID: %d) logged in".formatted(name, id);

// For simple concatenation, + or StringBuilder is fine
String greeting = "Hello, " + name;
```

---

## 5. Collections and Streams

### Immutable Collections

```java
// Factory methods (Java 9+)
List<String> names = List.of("Alice", "Bob", "Charlie");
Set<String> tags = Set.of("java", "spring");
Map<String, Integer> scores = Map.of("Alice", 95, "Bob", 87);

// Unmodifiable copies
List<String> copy = List.copyOf(mutableList);
```

### Stream Patterns

```java
// Transform
List<String> names = users.stream()
    .map(User::getName)
    .toList(); // Java 16+, replaces collect(Collectors.toList())

// Filter and collect
List<User> activeUsers = users.stream()
    .filter(User::isActive)
    .toList();

// Group by
Map<UserStatus, List<User>> grouped = users.stream()
    .collect(Collectors.groupingBy(User::getStatus));

// Aggregate
long totalAmount = orders.stream()
    .mapToLong(Order::getAmount)
    .sum();

// Find first
Optional<User> firstActive = users.stream()
    .filter(User::isActive)
    .findFirst();
```

### Sequenced Collections (Java 21+)

```java
// First and last element access
SequencedCollection<String> list = new ArrayList<>(List.of("a", "b", "c"));
String first = list.getFirst(); // "a"
String last = list.getLast();   // "c"

// Reversed view
SequencedCollection<String> reversed = list.reversed();

// SequencedMap
SequencedMap<String, Integer> map = new LinkedHashMap<>();
Map.Entry<String, Integer> firstEntry = map.firstEntry();
Map.Entry<String, Integer> lastEntry = map.lastEntry();
```

### Collection Rules

- Prefer `List.of()`, `Set.of()`, `Map.of()` for immutable collections
- Use `.toList()` (Java 16+) instead of `.collect(Collectors.toList())`
- Use `Stream` for transformation pipelines; avoid for simple iteration
- Avoid `Stream` for single-element operations — use direct method calls

---

## 6. Virtual Threads (Java 21+)

### Basic Usage

```java
// Create virtual thread
Thread.startVirtualThread(() -> {
    processRequest();
});

// Virtual thread executor
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> handleRequest(request));
}
```

### Spring Boot Integration

```yaml
# application.yml — enable virtual threads
spring:
  threads:
    virtual:
      enabled: true
```

```java
// Spring Boot 3.2+ with virtual threads
// No code changes needed — Tomcat/Jetty use virtual threads automatically
// when spring.threads.virtual.enabled=true

// Custom executor with virtual threads
@Bean
public AsyncTaskExecutor applicationTaskExecutor() {
    return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
}
```

### Virtual Thread Rules

- Virtual threads are cheap — create per-task, do not pool
- Blocking operations (JDBC, file I/O, `Thread.sleep`) are fine on virtual threads
- Avoid `synchronized` blocks in virtual thread code on Java 21–23 — use `ReentrantLock` instead
  (`synchronized` pins the carrier thread; resolved in Java 24+ via JEP 491)
- Do not use `ThreadLocal` for large objects — virtual threads may have millions of instances
- Virtual threads do not make CPU-bound work faster — they optimize I/O-bound concurrency

---

## 7. Naming Conventions

| Element        | Convention        | Example                          |
| -------------- | ----------------- | -------------------------------- |
| Class          | PascalCase        | `UserService`, `OrderResponse`   |
| Interface      | PascalCase        | `UserRepository`, `Cacheable`    |
| Method         | camelCase         | `findByEmail`, `calculateTotal`  |
| Variable       | camelCase         | `userName`, `isActive`           |
| Constant       | SCREAMING_SNAKE   | `MAX_RETRY_COUNT`, `DEFAULT_TTL` |
| Package        | lowercase dotted  | `com.example.user.service`       |
| Enum value     | SCREAMING_SNAKE   | `PENDING`, `IN_PROGRESS`         |
| Type parameter | Single uppercase  | `T`, `E`, `K`, `V`              |

---

## 8. Spring Boot Integration

### Constructor Injection

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    private final ApplicationEventPublisher eventPublisher;

    // Single constructor — @Autowired not needed
    public UserService(UserRepository userRepository,
                       ApplicationEventPublisher eventPublisher) {
        this.userRepository = userRepository;
        this.eventPublisher = eventPublisher;
    }
}
```

### Configuration Properties

```java
@ConfigurationProperties(prefix = "app.feature")
public record FeatureProperties(
    boolean enabled,
    int maxRetries,
    Duration timeout,
    List<String> allowedOrigins
) {
    public FeatureProperties {
        if (maxRetries < 0) throw new IllegalArgumentException("maxRetries must be >= 0");
        if (timeout == null) timeout = Duration.ofSeconds(30);
        if (allowedOrigins == null) allowedOrigins = List.of();
    }
}
```

### Controller Pattern

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse createUser(@Valid @RequestBody CreateUserRequest request) {
        return userService.create(request);
    }
}
```

### Entity Pattern (JPA)

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false, unique = true)
    private String email;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private UserStatus status = UserStatus.ACTIVE;

    @Column(nullable = false, updatable = false)
    private Instant createdAt = Instant.now();

    @Column(nullable = false)
    private Instant updatedAt = Instant.now();

    protected User() {} // JPA required

    public User(String name, String email) {
        this.name = Objects.requireNonNull(name);
        this.email = Objects.requireNonNull(email);
    }

    public void deactivate() {
        this.status = UserStatus.INACTIVE;
        this.updatedAt = Instant.now();
    }

    // Getters only — no setters for immutable fields
    public Long getId() { return id; }
    public String getName() { return name; }
    // ...
}
```

### Spring Boot Anti-Patterns

- Field injection with `@Autowired` — use constructor injection
- Returning JPA entities directly from controllers — use response DTOs
- `@Transactional` on private methods — Spring proxies cannot intercept
- Catching generic `Exception` in controllers — use `@ControllerAdvice`

---

## 9. Functional Interfaces and Lambdas

### Standard Functional Interfaces

| Interface           | Signature              | Use Case                     |
| ------------------- | ---------------------- | ---------------------------- |
| `Function<T, R>`    | `T → R`               | Transform input to output    |
| `Predicate<T>`      | `T → boolean`         | Test a condition             |
| `Consumer<T>`       | `T → void`            | Process without return       |
| `Supplier<T>`       | `() → T`              | Provide a value lazily       |
| `BiFunction<T,U,R>` | `(T, U) → R`          | Two-input transformation     |
| `UnaryOperator<T>`  | `T → T`               | Same-type transformation     |

### Lambda Best Practices

```java
// Prefer method references when possible
users.stream().map(User::getName)        // Good
users.stream().map(user -> user.getName()) // Acceptable but verbose

// Avoid complex lambdas — extract to named methods
users.stream()
    .filter(this::isEligibleForPromotion) // Clear intent
    .map(User::toResponse)
    .toList();

// Custom functional interface when standard ones are insufficient
@FunctionalInterface
public interface RetryableAction<T> {
    T execute() throws Exception;
}
```

---

## 10. Error Handling

### Exception Hierarchy

```java
// Base business exception
public abstract class BusinessException extends RuntimeException {
    private final ErrorCode errorCode;

    protected BusinessException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }

    protected BusinessException(ErrorCode errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    public ErrorCode getErrorCode() { return errorCode; }
}

// Specific exceptions
public class EntityNotFoundException extends BusinessException {
    public EntityNotFoundException(String entityName, Object id) {
        super(ErrorCode.ENTITY_NOT_FOUND,
            "%s not found: %s".formatted(entityName, id));
    }
}
```

### Try-With-Resources

```java
// Always use try-with-resources for AutoCloseable
try (var connection = dataSource.getConnection();
     var statement = connection.prepareStatement(sql)) {
    statement.setLong(1, userId);
    try (var resultSet = statement.executeQuery()) {
        // process results
    }
}

// var in try-with-resources (Java 9+)
var inputStream = new FileInputStream(file);
try (inputStream) {
    // use inputStream
}
```

---

## 11. Anti-Patterns

- Using raw types (`List` instead of `List<User>`)
- Returning `null` from collection methods — return `List.of()`
- Using `Optional` as a field, parameter, or collection element type
- Mutable public fields — use private fields with getters
- Using `Calendar` / `Date` — use `java.time` API (`LocalDate`, `Instant`, etc.)
- Using `StringBuffer` in single-threaded context — use `StringBuilder`
- Catching `Throwable` or `Error` — only catch `Exception` subtypes
- Empty catch blocks — at minimum log the exception
- Using `==` for String comparison — use `.equals()`
- `synchronized` with virtual threads on Java 21–23 — use `ReentrantLock` (fixed in Java 24+)
