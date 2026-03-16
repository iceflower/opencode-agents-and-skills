# JVM Concurrency Rules

## 1. Thread Safety Principles

### Core Rule: Minimize Shared Mutable State

| Strategy                 | Risk Level | Use Case                                |
| ------------------------ | ---------- | --------------------------------------- |
| Immutable objects        | Lowest     | Default choice for all shared data      |
| Thread-local confinement | Low        | Per-thread state (request context)      |
| Concurrent collections   | Medium     | Shared read-write data structures       |
| Explicit synchronization | High       | Complex invariants across multiple vars |

### Immutability First

```java
// Java — immutable record (thread-safe by design)
public record OrderSnapshot(
    Long orderId,
    BigDecimal amount,
    OrderStatus status,
    List<String> itemIds  // Must also be immutable
) {
    public OrderSnapshot {
        itemIds = List.copyOf(itemIds);  // Defensive copy
    }
}
```

```kotlin
// Kotlin — data class with val properties
data class OrderSnapshot(
    val orderId: Long,
    val amount: BigDecimal,
    val status: OrderStatus,
    val itemIds: List<String>  // Kotlin List is read-only
)
```

### Thread Safety Design Rules

- Make fields `final` / `val` by default — mutable fields require explicit justification
- Return unmodifiable collections from shared objects — never expose internal mutable state
- If a class is designed to be shared across threads, document it explicitly
- If a class is NOT thread-safe, document that too — do not leave ambiguity

---

## 2. JVM Memory Model Essentials

### Happens-Before Relationships

The Java Memory Model (JMM) defines **happens-before** as the guarantee that memory writes by one thread are visible to reads by another thread.

| Action A                       | Happens-Before Action B                |
| ------------------------------ | -------------------------------------- |
| `synchronized` block exit      | `synchronized` block entry (same lock) |
| `volatile` write               | `volatile` read (same variable)        |
| `Thread.start()` call          | First action in started thread         |
| Thread termination             | `Thread.join()` return                 |
| `Executor.submit()`            | Task execution start                   |
| `CountDownLatch.countDown()`   | `CountDownLatch.await()` return        |
| `CompletableFuture.complete()` | Dependent stage execution              |

### Visibility

```java
// Bad: no visibility guarantee — reader may never see updated value
class Broken {
    private boolean running = true;  // Not volatile, not synchronized
    void stop() { running = false; }
    void run() { while (running) { /* may loop forever */ } }
}

// Good: volatile guarantees visibility
class Fixed {
    private volatile boolean running = true;
    void stop() { running = false; }
    void run() { while (running) { /* sees update */ } }
}
```

### Atomicity

- Single reads/writes of `int`, `float`, `boolean`, references are atomic
- `long` and `double` are NOT atomic on 32-bit JVMs unless `volatile`
- Compound operations (`check-then-act`, `read-modify-write`) are NEVER atomic without synchronization

```java
// Bad: compound operation is not atomic
if (!map.containsKey(key)) {
    map.put(key, value);  // Race condition: another thread may insert between check and put
}

// Good: use atomic operation
map.putIfAbsent(key, value);
```

### Publication and Safe Construction

```java
// Bad: publishing reference before construction completes
class UnsafePublish {
    private static UnsafePublish instance;
    private final int value;

    public UnsafePublish() {
        instance = this;  // Escaping 'this' before constructor finishes
        value = 42;       // Another thread may see value = 0
    }
}

// Good: safe publication via volatile or final field
class SafePublish {
    private static volatile SafePublish instance;
    private final int value;

    public SafePublish() {
        value = 42;
        // Do not publish 'this' during construction
    }
}
```

---

## 3. Synchronization Tool Selection

### Decision Guide

| Requirement                        | Recommended Tool                                    |
| ---------------------------------- | --------------------------------------------------- |
| Simple mutual exclusion            | `synchronized` (Java) / `Mutex` (Kotlin coroutines) |
| Need tryLock, timed lock, fairness | `ReentrantLock`                                     |
| Read-heavy, write-rare             | `ReentrantReadWriteLock` or `StampedLock`           |
| Single atomic counter/flag         | `AtomicInteger`, `AtomicBoolean`, etc.              |
| Atomic reference swap              | `AtomicReference`, `VarHandle`                      |
| Multiple variables as single state | `synchronized` block or immutable snapshot          |
| High-contention counter            | `LongAdder` (not `AtomicLong`)                      |
| Thread-safe map                    | `ConcurrentHashMap`                                 |
| Thread-safe queue                  | `ConcurrentLinkedQueue`, `LinkedBlockingQueue`      |
| Coordinating threads               | `CountDownLatch`, `CyclicBarrier`, `Phaser`         |
| Semaphore-based access control     | `Semaphore`                                         |

### synchronized vs ReentrantLock

| Feature                 | `synchronized`       | `ReentrantLock`         |
| ----------------------- | -------------------- | ----------------------- |
| Automatic release       | Yes (block exit)     | No (must use `finally`) |
| Try-lock (non-blocking) | No                   | Yes (`tryLock()`)       |
| Timed lock              | No                   | Yes (`tryLock(time)`)   |
| Fairness policy         | No                   | Yes (constructor param) |
| Multiple conditions     | No (single wait set) | Yes (`newCondition()`)  |
| Virtual thread pinning  | Yes (Java 21-23)     | No                      |
| Performance             | Optimized by JVM     | Slightly more overhead  |

```java
// synchronized — simple, automatic release
synchronized (lock) {
    balance += amount;
}

// ReentrantLock — always use try-finally
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    balance += amount;
} finally {
    lock.unlock();  // Must always unlock
}
```

### Concurrent Collections

| Collection              | Thread-Safe Alternative         | Notes                           |
| ----------------------- | ------------------------------- | ------------------------------- |
| `HashMap`               | `ConcurrentHashMap`             | Lock striping, high concurrency |
| `TreeMap`               | `ConcurrentSkipListMap`         | Sorted, concurrent              |
| `ArrayList`             | `CopyOnWriteArrayList`          | Read-heavy, rare writes         |
| `HashSet`               | `ConcurrentHashMap.newKeySet()` | Concurrent set                  |
| `LinkedList` (as queue) | `ConcurrentLinkedQueue`         | Non-blocking queue              |
| `PriorityQueue`         | `PriorityBlockingQueue`         | Blocking priority queue         |

### Atomic Classes

```java
// AtomicInteger — lock-free counter
private final AtomicInteger requestCount = new AtomicInteger(0);
requestCount.incrementAndGet();

// LongAdder — better than AtomicLong under high contention
private final LongAdder totalBytes = new LongAdder();
totalBytes.add(bytesReceived);
long total = totalBytes.sum();

// AtomicReference — lock-free reference swap
private final AtomicReference<Config> config = new AtomicReference<>(initialConfig);
config.compareAndSet(oldConfig, newConfig);
```

---

## 4. Executor Framework

### ThreadPoolExecutor Configuration

```java
// Core parameters
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    corePoolSize,       // Threads kept alive even when idle
    maximumPoolSize,    // Maximum threads allowed
    keepAliveTime,      // Idle time before non-core threads terminate
    TimeUnit.SECONDS,
    workQueue,          // Queue for tasks when all core threads are busy
    threadFactory,      // Custom thread naming
    rejectionHandler    // Policy when pool and queue are full
);
```

### Pool Sizing Guidelines

| Workload Type | Formula                              | Rationale                       |
| ------------- | ------------------------------------ | ------------------------------- |
| CPU-bound     | `N_threads = N_cpu + 1`              | Minimal context switching       |
| I/O-bound     | `N_threads = N_cpu * (1 + W/C)`      | W = wait time, C = compute time |
| Mixed         | Separate pools for CPU and I/O tasks | Prevents I/O blocking CPU work  |

- `N_cpu = Runtime.getRuntime().availableProcessors()`
- For typical web applications (I/O-bound): 2x-4x CPU core count
- Always benchmark with realistic load — formulas are starting points

### Work Queue Selection

| Queue Type              | Behavior                        | Use Case                       |
| ----------------------- | ------------------------------- | ------------------------------ |
| `SynchronousQueue`      | No buffering, direct handoff    | Cached thread pool             |
| `LinkedBlockingQueue`   | Unbounded (or bounded) FIFO     | Fixed thread pool              |
| `ArrayBlockingQueue`    | Bounded FIFO, contiguous memory | Bounded pool with backpressure |
| `PriorityBlockingQueue` | Priority ordering               | Priority-based scheduling      |

### Rejection Policies

| Policy                | Behavior                           | Use Case                        |
| --------------------- | ---------------------------------- | ------------------------------- |
| `AbortPolicy`         | Throw `RejectedExecutionException` | Default, fail-fast              |
| `CallerRunsPolicy`    | Execute in caller's thread         | Natural backpressure            |
| `DiscardPolicy`       | Silently discard                   | Lossy is acceptable             |
| `DiscardOldestPolicy` | Discard oldest queued task         | Newest task has higher priority |

### Executor Best Practices

```java
// Named thread factory for debugging
ThreadFactory factory = Thread.ofPlatform()
    .name("order-processor-", 0)
    .daemon(true)
    .factory();

// Graceful shutdown
executor.shutdown();
if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
    executor.shutdownNow();
}
```

- Always name threads — unnamed threads are impossible to debug in thread dumps
- Always shut down executors — leaked executors prevent JVM shutdown
- Never use `Executors.newCachedThreadPool()` with unbounded task submission — can create thousands of threads
- Never use `Executors.newFixedThreadPool()` with unbounded queue for latency-sensitive work — queue grows without bound

---

## 5. CompletableFuture Patterns

### Composition

```java
// Sequential: thenApply, thenCompose
CompletableFuture<UserProfile> profile = fetchUser(userId)
    .thenCompose(user -> fetchProfile(user.getProfileId()))
    .thenApply(this::enrichProfile);

// Parallel: allOf, anyOf
CompletableFuture<UserProfile> profileFuture = fetchProfile(userId);
CompletableFuture<List<Order>> ordersFuture = fetchOrders(userId);
CompletableFuture<Notifications> notifsFuture = fetchNotifications(userId);

CompletableFuture<Dashboard> dashboard = CompletableFuture
    .allOf(profileFuture, ordersFuture, notifsFuture)
    .thenApply(v -> new Dashboard(
        profileFuture.join(),
        ordersFuture.join(),
        notifsFuture.join()
    ));

// Combine two futures
CompletableFuture<Summary> summary = profileFuture
    .thenCombine(ordersFuture, (profile, orders) -> new Summary(profile, orders));
```

### Error Handling

```java
// exceptionally — recover from errors
CompletableFuture<User> user = fetchUser(userId)
    .exceptionally(ex -> {
        log.warn("Failed to fetch user, using fallback", ex);
        return User.fallback(userId);
    });

// handle — process both success and failure
CompletableFuture<Result> result = fetchData()
    .handle((data, ex) -> {
        if (ex != null) return Result.failure(ex);
        return Result.success(data);
    });

// whenComplete — side effect without changing result
fetchData()
    .whenComplete((data, ex) -> {
        if (ex != null) log.error("Fetch failed", ex);
        else log.info("Fetched: {}", data);
    });
```

### Timeout (Java 9+)

```java
CompletableFuture<Response> response = callExternalApi()
    .orTimeout(5, TimeUnit.SECONDS)                    // Fails with TimeoutException
    .exceptionally(ex -> Response.timeout());

CompletableFuture<Response> response = callExternalApi()
    .completeOnTimeout(Response.fallback(), 5, TimeUnit.SECONDS);  // Returns default
```

### CompletableFuture Rules

- Always specify executor for async stages — default `ForkJoinPool.commonPool()` is shared globally
- Never call `join()` or `get()` on event loop or request-handling threads — causes blocking
- Use `thenCompose` (not `thenApply`) when the mapping function returns a `CompletableFuture`
- Handle exceptions at every stage that can fail — unhandled exceptions are silently swallowed
- Use `orTimeout` or `completeOnTimeout` — never wait indefinitely

```java
// Always provide executor for async operations
CompletableFuture<Data> data = CompletableFuture
    .supplyAsync(() -> fetchData(), ioExecutor)        // Explicit executor
    .thenApplyAsync(this::transform, computeExecutor); // Different pool for CPU work
```

---

## 6. Kotlin Coroutines In-Depth

### Dispatcher Selection

| Dispatcher                         | Thread Pool      | Use Case                              |
| ---------------------------------- | ---------------- | ------------------------------------- |
| `Dispatchers.Default`              | CPU core count   | CPU-bound computation                 |
| `Dispatchers.IO`                   | Up to 64 threads | Blocking I/O (JDBC, file, legacy SDK) |
| `Dispatchers.Main`                 | Main/UI thread   | Android UI updates                    |
| `Dispatchers.Unconfined`           | Caller's thread  | Testing only — never in production    |
| Custom `newFixedThreadPoolContext` | Configurable     | Isolated work for specific service    |

```kotlin
// CPU-bound work
withContext(Dispatchers.Default) {
    computeExpensiveResult()
}

// Blocking I/O
withContext(Dispatchers.IO) {
    jdbcTemplate.query("SELECT * FROM users")
}

// Custom limited dispatcher (throttle concurrency)
val dbDispatcher = Dispatchers.IO.limitedParallelism(10)
withContext(dbDispatcher) {
    repository.save(entity)
}
```

### Structured Concurrency

```kotlin
// coroutineScope — all-or-nothing: any child failure cancels all siblings
suspend fun fetchDashboard(userId: Long): Dashboard = coroutineScope {
    val profile = async { userService.getProfile(userId) }
    val orders = async { orderService.getRecentOrders(userId) }

    Dashboard(profile.await(), orders.await())
    // If either fails, the other is cancelled automatically
}

// supervisorScope — child failures do NOT cancel siblings
suspend fun fetchDashboardWithFallback(userId: Long): Dashboard = supervisorScope {
    val profile = async { userService.getProfile(userId) }
    val orders = async {
        try { orderService.getRecentOrders(userId) }
        catch (e: Exception) { emptyList() }  // Fallback — does not cancel profile
    }

    Dashboard(profile.await(), orders.await())
}
```

### Exception Propagation

```kotlin
// Launch: exceptions propagate to parent (crash scope)
scope.launch {
    throw RuntimeException("Crash")  // Parent scope is cancelled
}

// Async: exceptions are deferred to await()
val deferred = scope.async {
    throw RuntimeException("Deferred crash")
}
try {
    deferred.await()  // Exception thrown here
} catch (e: RuntimeException) {
    // Handle here
}
```

### Coroutine Exception Handler

```kotlin
// Top-level exception handler (last resort, not for recovery)
val handler = CoroutineExceptionHandler { _, exception ->
    log.error("Uncaught coroutine exception", exception)
}

val scope = CoroutineScope(SupervisorJob() + handler)
scope.launch {
    throw RuntimeException("Handled by CoroutineExceptionHandler")
}
```

### Cancellation Rules

```kotlin
// CancellationException is special — do NOT catch it
suspend fun process() {
    try {
        longRunningWork()
    } catch (e: CancellationException) {
        throw e  // Must rethrow — swallowing breaks structured concurrency
    } catch (e: Exception) {
        log.error("Processing failed", e)
    }
}

// Check for cancellation in CPU-bound loops
suspend fun computeHeavy(data: List<Item>) {
    for (item in data) {
        ensureActive()  // Throws CancellationException if cancelled
        process(item)
    }
}
```

### Coroutine Rules

- Never use `GlobalScope` — it breaks structured concurrency and causes memory leaks
- Never use `runBlocking` in request handlers — it blocks the calling thread
- Always use `supervisorScope` when child failures should be independent
- Always call `ensureActive()` or `yield()` in CPU-bound loops for cancellation support
- Use `withTimeout` for coroutine-level timeout, not `Thread.sleep`-based approaches

---

## 7. Virtual Threads (Java 21+)

### When to Use Virtual Threads

| Scenario                         | Virtual Threads | Platform Threads |
| -------------------------------- | --------------- | ---------------- |
| I/O-bound (HTTP calls, DB, file) | Yes             |                  |
| CPU-bound computation            |                 | Yes              |
| High concurrency (10K+ tasks)    | Yes             |                  |
| Legacy code with `synchronized`  | Caution         | Yes              |
| Thread-per-request web server    | Yes             |                  |

### Basic Usage

```java
// Create virtual thread
Thread.startVirtualThread(() -> handleRequest(request));

// Executor for virtual threads — one thread per task, no pooling
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (Request req : requests) {
        executor.submit(() -> handleRequest(req));
    }
}

// Spring Boot 3.2+ — configuration only
// application.yml: spring.threads.virtual.enabled=true
```

### Virtual Thread Caveats

```java
// Caveat 1: synchronized pins the carrier thread (Java 21-23)
// Fixed in Java 24+ (JEP 491)
synchronized (lock) {
    blockingIoCall();  // Pins carrier thread — avoid on Java 21-23
}

// Fix: use ReentrantLock instead (Java 21-23)
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    blockingIoCall();  // Does NOT pin carrier thread
} finally {
    lock.unlock();
}

// Caveat 2: Do NOT pool virtual threads — they are cheap, create per task
// Bad: pooling virtual threads defeats their purpose
ExecutorService pool = Executors.newFixedThreadPool(10,
    Thread.ofVirtual().factory());  // Wrong — do not pool

// Good: one virtual thread per task
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

// Caveat 3: ThreadLocal overhead — each virtual thread has its own copy
// Millions of virtual threads = millions of ThreadLocal copies
// Use ScopedValue (preview) instead for large-scale virtual thread workloads
```

### Virtual Thread Rules

- Virtual threads are cheap — create per task, never pool them
- Blocking operations (JDBC, `Thread.sleep`, file I/O) are fine on virtual threads
- Avoid `synchronized` on Java 21-23 — use `ReentrantLock` (fixed in Java 24+)
- Avoid `ThreadLocal` with large objects — prefer `ScopedValue` (preview)
- Virtual threads do NOT speed up CPU-bound work — they optimize I/O concurrency
- Do not set thread priority on virtual threads — it has no effect
- Do not use `Thread.yield()` — virtual threads yield automatically on blocking

---

## 8. Concurrency Bug Patterns

### Race Condition

```java
// Bug: check-then-act without synchronization
if (map.containsKey(key)) {
    return map.get(key);  // Another thread may remove between check and get
}

// Fix: use atomic operation
return map.computeIfAbsent(key, k -> createValue(k));
```

### Deadlock

```java
// Bug: inconsistent lock ordering
// Thread 1: lock(A) → lock(B)
// Thread 2: lock(B) → lock(A)
synchronized (accountA) {
    synchronized (accountB) {
        transfer(accountA, accountB, amount);
    }
}

// Fix: always acquire locks in consistent global order
Account first = accountA.getId() < accountB.getId() ? accountA : accountB;
Account second = accountA.getId() < accountB.getId() ? accountB : accountA;
synchronized (first) {
    synchronized (second) {
        transfer(accountA, accountB, amount);
    }
}
```

### Starvation

```java
// Bug: unfair lock with long-holding writer starves readers
// Readers never get lock because writers keep acquiring it

// Fix: use fair lock or ReadWriteLock
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock(true);  // fair = true
```

### Livelock

```java
// Bug: two threads keep yielding to each other without progress
// Thread 1: "I see Thread 2 is working, I'll back off"
// Thread 2: "I see Thread 1 is working, I'll back off"

// Fix: add randomized backoff to break symmetry
Thread.sleep(ThreadLocalRandom.current().nextInt(10, 100));
```

### Data Race vs Race Condition

| Term           | Definition                                                              | Example                             |
| -------------- | ----------------------------------------------------------------------- | ----------------------------------- |
| Data race      | Two threads access same memory, at least one writes, no synchronization | Reading `long` while another writes |
| Race condition | Correctness depends on execution order                                  | Check-then-act on shared state      |

- Data races are always bugs — they violate the JMM
- Race conditions are logic bugs — they may work most of the time but fail under contention
- Fixing data races (adding `volatile` / `synchronized`) does NOT always fix race conditions

---

## 9. Concurrency Testing

### Challenges

- Concurrent bugs are non-deterministic — they may not reproduce on every run
- Thread scheduling varies across runs, JVMs, and hardware
- A test passing 1000 times does not guarantee correctness

### Testing Strategies

| Strategy                 | Tool / Technique                   | Catches                         |
| ------------------------ | ---------------------------------- | ------------------------------- |
| Stress testing           | Multiple threads hitting same code | Race conditions under load      |
| Deterministic scheduling | `jcstress`, `Lincheck`             | All possible interleavings      |
| Static analysis          | SpotBugs, IntelliJ inspections     | Common concurrency antipatterns |
| Thread dump analysis     | `jstack`, `jcmd`                   | Deadlocks, thread starvation    |
| Assertions with latches  | `CountDownLatch`, `CyclicBarrier`  | Ordering violations             |

### JCStress Example

```java
@JCStressTest
@Outcome(id = "1, 1", expect = Expect.ACCEPTABLE, desc = "Both see update")
@Outcome(id = "0, 0", expect = Expect.ACCEPTABLE, desc = "Neither sees update")
@Outcome(id = "1, 0", expect = Expect.ACCEPTABLE, desc = "Only x seen")
@Outcome(id = "0, 1", expect = Expect.ACCEPTABLE_INTERESTING, desc = "Reordering observed")
@State
public class ReorderingTest {
    int x, y;

    @Actor
    public void writer() { x = 1; y = 1; }

    @Actor
    public void reader(II_Result r) { r.r1 = y; r.r2 = x; }
}
```

### Lincheck Example (Kotlin)

```kotlin
class ConcurrentMapTest {
    private val map = ConcurrentHashMap<Int, Int>()

    @Operation
    fun put(key: Int, value: Int) = map.put(key, value)

    @Operation
    fun get(key: Int) = map.get(key)

    @Test
    fun stressTest() = StressOptions().check(this::class)

    @Test
    fun modelCheckingTest() = ModelCheckingOptions().check(this::class)
}
```

### CountDownLatch Test Pattern

```java
@Test
void shouldBeThreadSafe() throws InterruptedException {
    int threadCount = 100;
    CountDownLatch startLatch = new CountDownLatch(1);
    CountDownLatch doneLatch = new CountDownLatch(threadCount);
    AtomicInteger errorCount = new AtomicInteger(0);

    for (int i = 0; i < threadCount; i++) {
        Thread.startVirtualThread(() -> {
            try {
                startLatch.await();  // All threads start simultaneously
                // Exercise the code under test
                service.process();
            } catch (Exception e) {
                errorCount.incrementAndGet();
            } finally {
                doneLatch.countDown();
            }
        });
    }

    startLatch.countDown();  // Release all threads at once
    doneLatch.await(10, TimeUnit.SECONDS);
    assertThat(errorCount.get()).isZero();
}
```

### Coroutine Testing

```kotlin
@Test
fun `concurrent access should be safe`() = runTest {
    val counter = AtomicInteger(0)
    val jobs = (1..1000).map {
        launch(Dispatchers.Default) {
            service.incrementSafely(counter)
        }
    }
    jobs.forEach { it.join() }
    assertThat(counter.get()).isEqualTo(1000)
}
```

---

## 10. Anti-Patterns

### Synchronization Anti-Patterns

```java
// Bad: synchronizing on mutable field
private Object lock = new Object();  // Can be reassigned
synchronized (lock) { ... }

// Good: use final lock object
private final Object lock = new Object();

// Bad: synchronizing on boxed primitive (shared instance)
synchronized (Integer.valueOf(42)) { ... }  // Shared across JVM

// Bad: synchronizing on String literal (interned, shared)
synchronized ("lock") { ... }

// Good: dedicated private lock object
private final Object lock = new Object();
synchronized (lock) { ... }
```

### Thread Pool Anti-Patterns

```java
// Bad: creating thread per task without limit
new Thread(() -> handleRequest(req)).start();  // Unbounded thread creation

// Bad: unbounded cached pool for unpredictable workload
Executors.newCachedThreadPool();  // Can create Integer.MAX_VALUE threads

// Good: bounded pool with rejection policy
new ThreadPoolExecutor(
    10, 50, 60L, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(100),
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

### Other Anti-Patterns

| Anti-Pattern                                       | Problem                                      | Fix                                                     |
| -------------------------------------------------- | -------------------------------------------- | ------------------------------------------------------- |
| `synchronized` everywhere                          | Reduces concurrency, risk of deadlock        | Use concurrent collections or atomic types              |
| Double-checked locking without `volatile`          | Broken under JMM — partially constructed obj | Add `volatile` or use `Lazy` / holder                   |
| `Thread.stop()` / `Thread.suspend()`               | Deprecated — corrupts state                  | Use interrupt-based cancellation                        |
| `ThreadLocal` with thread pools                    | Values leak across tasks if not cleaned      | Always `remove()` in `finally`                          |
| `ThreadLocal` with virtual threads                 | Millions of copies — memory waste            | Use `ScopedValue` (preview)                             |
| Catching `InterruptedException` silently           | Breaks cancellation protocol                 | Restore interrupt: `Thread.currentThread().interrupt()` |
| `synchronized` in virtual thread code (Java 21-23) | Pins carrier thread                          | Use `ReentrantLock` (fixed in Java 24+)                 |
| `runBlocking` in request handlers                  | Blocks calling thread entirely               | Use `suspend fun` or `CompletableFuture`                |
| `GlobalScope.launch`                               | No structured concurrency, memory leaks      | Use `coroutineScope` or `supervisorScope`               |
| Polling with `Thread.sleep` in a loop              | Wastes thread, delays response               | Use `wait/notify`, `Condition`, or async                |
| Ignoring `Future.get()` exceptions                 | Swallowed failures, silent data loss         | Always handle or propagate exceptions                   |

---

## 11. Related Rules

- **Java conventions**: see `java-convention.md` (virtual threads, sealed classes, records)
- **Kotlin conventions**: see `kotlin-convention.md` (coroutines basics, extension functions)
- **Spring WebFlux and Coroutines**: see `spring-webflux.md` (non-blocking, Flow, R2DBC)
- **JVM performance**: see `jvm-performance.md` (GC tuning, profiling, memory layout)
- **Spring Framework**: see `spring-framework.md` (`@Async`, `@Scheduled`, event system)

---

## 12. Further Reading

| Topic                    | Resource                                                  |
| ------------------------ | --------------------------------------------------------- |
| JVM memory model         | JSR-133 FAQ, JLS Chapter 17                               |
| Concurrency fundamentals | *Java Concurrency in Practice* (Goetz et al.)             |
| Virtual threads          | JEP 444, JEP 491 (synchronized pinning fix)               |
| Kotlin coroutines        | kotlinx.coroutines guide (official)                       |
| Lock-free algorithms     | *The Art of Multiprocessor Programming* (Herlihy, Shavit) |
| Concurrency testing      | jcstress (OpenJDK), Lincheck (JetBrains)                  |
| Structured concurrency   | JEP 453 (Java preview), Kotlin coroutines documentation   |
