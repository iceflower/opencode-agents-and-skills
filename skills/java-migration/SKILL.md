---
name: java-migration
description: >-
  Java version migration guide and breaking changes reference.
  Use when upgrading Java versions, migrating javax to jakarta,
  adopting records/sealed classes/virtual threads, or reviewing
  Spring Boot version compatibility.
---

# Java Version Migration Rules

## 1. Version Overview

| Version | Release    | LTS | Key Theme                                          |
| ------- | ---------- | --- | -------------------------------------------------- |
| 8       | 2014-03    | Yes | Lambdas, Streams, Optional, java.time              |
| 11      | 2018-09    | Yes | HTTP Client, var, String methods                   |
| 14      | 2020-03    |     | Switch expressions, Records (preview)              |
| 15      | 2020-09    |     | Text blocks, Sealed classes (preview)              |
| 16      | 2021-03    |     | Records (stable), Pattern matching instanceof      |
| 17      | 2021-09    | Yes | Sealed classes (stable), Pattern matching switch (preview) |
| 21      | 2023-09    | Yes | Virtual threads, Pattern matching switch, Sequenced collections |
| 24      | 2025-03    |     | Stream gatherers, Flexible constructor bodies      |
| 25      | 2025-09    | Yes | Compact source files, Module imports, Scoped values |

### LTS Upgrade Path

```text
Java 8 → Java 11 → Java 17 → Java 21 → Java 25 (latest LTS)
```

---

## 2. Java 8 → 11

### Local Variable Type Inference (var)

```java
// var for local variables with initializers
var users = new ArrayList<User>();      // ArrayList<User>
var name = user.getName();              // String
var stream = users.stream();            // Stream<User>

// var in lambda parameters (Java 11)
users.stream()
    .map((@NonNull var user) -> user.getName())
    .collect(Collectors.toList());
```

### var Rules

| Use var                          | Avoid var                              |
| -------------------------------- | -------------------------------------- |
| Type is obvious from right side  | Type is unclear without declaration    |
| Long generic types               | Primitive types where precision matters|
| Try-with-resources               | Return type is not obvious             |

### New String Methods

```java
"  hello  ".strip();          // "hello" (Unicode-aware, unlike trim())
"  hello  ".stripLeading();   // "hello  "
"  hello  ".stripTrailing();  // "  hello"
"  ".isBlank();               // true
"line1\nline2".lines();       // Stream<String>
"ha".repeat(3);               // "hahaha"
```

### HTTP Client (java.net.http)

```java
var client = HttpClient.newHttpClient();
var request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users"))
    .header("Content-Type", "application/json")
    .GET()
    .build();

HttpResponse<String> response = client.send(request,
    HttpResponse.BodyHandlers.ofString());
```

### Files Utility Methods

```java
// Read/write entire file
String content = Files.readString(Path.of("file.txt"));
Files.writeString(Path.of("output.txt"), content);
```

### Migration Notes

- Remove JavaEE modules from classpath — `javax.xml.bind`, `javax.annotation` moved to Jakarta
- Add explicit dependencies: `jakarta.xml.bind-api`, `jakarta.annotation-api`
- JavaFX removed from JDK — add as separate dependency if needed
- Nashorn JavaScript engine deprecated (removed in Java 15)

---

## 3. Java 11 → 17

### Switch Expressions (Java 14+)

```java
// Before: switch statement
String label;
switch (status) {
    case ACTIVE:
        label = "Active";
        break;
    case INACTIVE:
        label = "Inactive";
        break;
    default:
        label = "Unknown";
}

// After: switch expression with arrow syntax
String label = switch (status) {
    case ACTIVE -> "Active";
    case INACTIVE -> "Inactive";
    default -> "Unknown";
};

// Multi-value cases
String category = switch (dayOfWeek) {
    case MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY -> "Weekday";
    case SATURDAY, SUNDAY -> "Weekend";
};

// Block body with yield
int value = switch (input) {
    case "a" -> 1;
    case "b" -> {
        log.info("Processing b");
        yield 2;
    }
    default -> 0;
};
```

### Records (Java 16+)

```java
// Immutable data carrier
public record Point(int x, int y) {}

// With compact constructor
public record Range(int start, int end) {
    public Range {
        if (start > end) throw new IllegalArgumentException("start > end");
    }
}

// Implementing interfaces
public record ApiResponse<T>(T data, ResponseMeta meta)
    implements Serializable {}
```

### Pattern Matching for instanceof (Java 16+)

```java
// Before
if (obj instanceof String) {
    String s = (String) obj;
    return s.length();
}

// After
if (obj instanceof String s) {
    return s.length();
}
```

### Sealed Classes (Java 17+)

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {
    double area();
}

public record Circle(double radius) implements Shape {
    public double area() { return Math.PI * radius * radius; }
}
public record Rectangle(double width, double height) implements Shape {
    public double area() { return width * height; }
}
public record Triangle(double base, double height) implements Shape {
    public double area() { return 0.5 * base * height; }
}
```

### Text Blocks (Java 15+)

```java
String json = """
    {
        "name": "John",
        "email": "john@example.com"
    }
    """;

String sql = """
    SELECT u.id, u.name
    FROM users u
    WHERE u.status = 'ACTIVE'
    """;
```

### Helpful NullPointerException (Java 14+)

```java
// JVM now shows which variable was null
// Before: NullPointerException
// After:  NullPointerException: Cannot invoke "String.length()"
//         because the return value of "User.getName()" is null
```

### Other Notable Additions

```java
// Stream.toList() — Java 16+
List<String> names = users.stream().map(User::getName).toList();

// Collectors.teeing() — Java 12+
var result = users.stream().collect(
    Collectors.teeing(
        Collectors.counting(),
        Collectors.averagingInt(User::getAge),
        (count, avgAge) -> new UserStats(count, avgAge)
    )
);

// String methods — Java 12+
"  hello  ".indent(4);        // Adds 4 spaces to each line
"Hello".transform(s -> s + "!"); // Applies function
```

### Migration Notes

- `javax.*` packages migrating to `jakarta.*` (relevant for Spring Boot 3.x)
- Strong encapsulation of JDK internals — `--add-opens` may be needed for reflection
- Removed: Nashorn, RMI Activation, Applet API
- SecurityManager deprecated for removal
- GC: G1 is default; ZGC and Shenandoah available for low-latency workloads

---

## 4. Java 17 → 21

### Virtual Threads (Project Loom)

```java
// Create virtual threads
Thread.startVirtualThread(() -> processRequest());

// Virtual thread executor
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i ->
        executor.submit(() -> handleRequest(i))
    );
}

// Thread.Builder API
Thread thread = Thread.ofVirtual()
    .name("worker-", 0)
    .start(() -> doWork());
```

### Pattern Matching for Switch (Stable)

```java
// Type patterns with guards
public String format(Object obj) {
    return switch (obj) {
        case Integer i when i > 0 -> "positive: " + i;
        case Integer i            -> "non-positive: " + i;
        case String s             -> "string: " + s;
        case null                 -> "null";
        default                   -> obj.toString();
    };
}
```

### Record Patterns

```java
// Destructuring in switch and instanceof
record Pair<A, B>(A first, B second) {}

if (obj instanceof Pair<String, Integer>(var name, var age)) {
    System.out.println(name + ": " + age);
}

// Nested patterns
record Address(String city, String zip) {}
record Customer(String name, Address address) {}

String city = switch (customer) {
    case Customer(var n, Address(var c, var z)) -> c;
};
```

### Sequenced Collections

```java
// New interfaces: SequencedCollection, SequencedSet, SequencedMap
SequencedCollection<String> list = new ArrayList<>();
list.addFirst("first");
list.addLast("last");
String first = list.getFirst();
String last = list.getLast();
SequencedCollection<String> reversed = list.reversed();

// SequencedMap
SequencedMap<String, Integer> map = new LinkedHashMap<>();
map.putFirst("a", 1);
map.putLast("z", 26);
```

### Scoped Values (Java 25+)

Scoped Values are a new model for thread-local variables, adapted to virtual threads. They became a stable feature in Java 25 (JEP 506).

```java
// Replacement for ThreadLocal in virtual thread context
private static final ScopedValue<UserContext> CONTEXT = ScopedValue.newInstance();

ScopedValue.runWhere(CONTEXT, userContext, () -> {
    // CONTEXT.get() available in this scope and child threads
    processRequest();
});
```

### Structured Concurrency (Preview)

Structured Concurrency (JEP 505) is still in preview as of Java 25. It simplifies concurrent programming by treating groups of related tasks as a single unit of work.

```java
// Still preview in Java 25 - requires --enable-preview
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Supplier<String> user = scope.fork(() -> findUser());
    Supplier<Integer> order = scope.fork(() -> fetchOrder());
    
    scope.join();
    scope.throwIfFailed();
    
    return new Response(user.get(), order.get());
}
```

### Migration Notes

- Virtual threads: replace thread pools with `Executors.newVirtualThreadPerTaskExecutor()` for I/O-bound tasks
- Replace `synchronized` with `ReentrantLock` in virtual thread code paths (Java 21–23; resolved in Java 24+ via JEP 491)
- Replace `ThreadLocal` with `ScopedValue` where possible (stable since Java 25)
- Spring Boot 3.2+ supports virtual threads via `spring.threads.virtual.enabled=true`
- Spring Boot 4.0+ supports Java 25 with first-class support
- Deprecation: finalization (`finalize()` method) — use `Cleaner` or try-with-resources

---

## 5. Java 21 → 24

### Stream Gatherers (Java 24+)

```java
// Custom intermediate stream operations
List<List<Integer>> windows = numbers.stream()
    .gather(Gatherers.windowFixed(3))
    .toList();
// [1,2,3] → [[1,2,3], [4,5,6], ...]

// Sliding window
List<List<Integer>> sliding = numbers.stream()
    .gather(Gatherers.windowSliding(3))
    .toList();
// [1,2,3,4] → [[1,2,3], [2,3,4]]

// Fold gatherer — reduces all elements into a single result
Gatherer<Integer, ?, Integer> sum = Gatherers.fold(
    () -> 0,
    (acc, element) -> acc + element
);
```

### Flexible Constructor Bodies (Java 24+)

```java
// Statements before super() or this() call
public class ValidatedUser extends User {
    public ValidatedUser(String name, String email) {
        // Validation before super() — previously not allowed
        Objects.requireNonNull(name, "name must not be null");
        if (name.isBlank()) throw new IllegalArgumentException("name is blank");
        super(name, email);
    }
}
```

### Unnamed Variables and Patterns (Java 22+)

```java
// Use _ for unused variables
try {
    processData();
} catch (NumberFormatException _) {
    // Exception variable not used
    return defaultValue;
}

// In enhanced for
for (var _ : collection) {
    count++;
}

// In lambda
map.forEach((_, value) -> process(value));

// In pattern matching
if (obj instanceof Point(var x, _)) {
    // Only need x coordinate
}
```

### Migration Notes

- Stream Gatherers replace many custom collector patterns — review existing `Collector` implementations
- Unnamed variables (`_`) improve readability — adopt gradually in new code
- Flexible constructor bodies simplify validation in inheritance hierarchies

---

## 6. Migration Checklist

### Before Version Upgrade

- [ ] Check deprecated API usage (`jdeprscan` tool)
- [ ] Verify library/framework compatibility with target Java version
- [ ] Check for JDK internal API usage (`jdeps --jdk-internals`)
- [ ] Review `--add-opens` / `--add-exports` flags needed
- [ ] Verify build tool support (Maven/Gradle version)

### Upgrade Procedure

1. Update JDK version in build tool configuration
2. Update `sourceCompatibility` and `targetCompatibility`
3. Run clean build — fix compile errors
4. Fix deprecation warnings
5. Run full test suite
6. Validate in CI pipeline

### After Version Upgrade

- [ ] Adopt new language features in new code (records, pattern matching, etc.)
- [ ] Replace deprecated APIs (see table below)
- [ ] Consider virtual threads for I/O-bound workloads (Java 21+)
- [ ] Review GC settings for new JDK defaults

---

## 7. Deprecated API Replacement Guide

| Deprecated                           | Replacement                           | Since |
| ------------------------------------ | ------------------------------------- | ----- |
| `new Integer(42)`                    | `Integer.valueOf(42)`                 | 9     |
| `new String("hello")`               | `"hello"` (literal)                   | -     |
| `Thread.stop()` / `suspend()`       | Cooperative cancellation              | 2     |
| `finalize()`                         | `Cleaner` or try-with-resources       | 9     |
| `SecurityManager`                    | No direct replacement — redesign      | 17    |
| `javax.*` (EE packages)             | `jakarta.*`                           | 11    |
| `Date` / `Calendar`                 | `java.time.LocalDate`, `Instant`      | 8     |
| `Vector` / `Hashtable`              | `ArrayList` / `HashMap`               | 2     |
| `StringBuffer` (single-threaded)    | `StringBuilder`                       | 5     |
| `URL.openConnection()` (HTTP)       | `java.net.http.HttpClient`            | 11    |
| `Collectors.toList()` (mutable)     | `Stream.toList()` (unmodifiable)      | 16    |
| `ThreadLocal` (virtual threads)     | `ScopedValue` (stable since Java 25)  | 25    |
| `synchronized` (with virtual threads) | Prefer `ReentrantLock` on Java 21–23 (pinning fixed in 24+) | 21 |

---

## 8. Spring Boot Version Compatibility

For detailed Spring Boot migration guide (2.7 → 3.x → 4.0), see `spring-boot-migration.md`.

| Spring Boot | Minimum Java | Recommended Java | Jakarta EE    |
| ----------- | ------------ | ---------------- | ------------- |
| 2.7.x       | 8            | 17               | javax         |
| 3.0.x       | 17           | 17               | jakarta (EE 9)|
| 3.1–3.4     | 17           | 21               | jakarta (EE 9)|
| 4.0.x       | 17           | 25               | jakarta (EE 11)|
