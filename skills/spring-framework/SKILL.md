---
name: spring-framework
description: >-
  Spring Framework core conventions including IoC/DI, AOP, transaction management,
  event system, bean lifecycle, MVC interceptors, validation, and scheduling.
  Use when working with Spring Framework core features.
---

# Spring Framework Core Rules

## 1. IoC and Dependency Injection

### Constructor Injection (Preferred)

```java
@Service
public class OrderService {
    private final UserRepository userRepository;
    private final PaymentGateway paymentGateway;
    private final ApplicationEventPublisher eventPublisher;

    // Single constructor — no @Autowired needed
    public OrderService(UserRepository userRepository,
                        PaymentGateway paymentGateway,
                        ApplicationEventPublisher eventPublisher) {
        this.userRepository = userRepository;
        this.paymentGateway = paymentGateway;
        this.eventPublisher = eventPublisher;
    }
}
```

```kotlin
// Kotlin — primary constructor injection
@Service
class OrderService(
    private val userRepository: UserRepository,
    private val paymentGateway: PaymentGateway,
    private val eventPublisher: ApplicationEventPublisher
)
```

### Injection Anti-Patterns

```java
// Bad: field injection — hides dependencies, untestable without reflection
@Autowired
private UserRepository userRepository;

// Bad: setter injection — mutable dependency, easy to forget
@Autowired
public void setUserRepository(UserRepository repo) { ... }

// Bad: multiple constructors without @Autowired — ambiguous
public OrderService(UserRepository repo) { ... }
public OrderService(UserRepository repo, PaymentGateway gw) { ... }
```

### Bean Scope

| Scope       | Lifecycle                        | Use Case                         |
| ----------- | -------------------------------- | -------------------------------- |
| `singleton` | One instance per ApplicationContext (default) | Stateless services, repositories |
| `prototype` | New instance per injection/request | Stateful, short-lived objects    |
| `request`   | One per HTTP request             | Request-scoped data              |
| `session`   | One per HTTP session             | Session-scoped data              |

- Default is `singleton` — do not store mutable state in singleton beans
- Injecting `prototype` into `singleton` does NOT create new instances — use `ObjectProvider<T>` or `@Lookup`

### Conditional Bean Registration

```java
@Configuration
public class InfraConfig {
    @Bean
    @Profile("production")
    public DataSource productionDataSource() { ... }

    @Bean
    @Profile("local")
    public DataSource localDataSource() { ... }

    @Bean
    @ConditionalOnProperty(name = "app.cache.enabled", havingValue = "true")
    public CacheManager cacheManager() { ... }
}
```

---

## 2. AOP (Aspect-Oriented Programming)

### Aspect Definition

```java
@Aspect
@Component
public class LoggingAspect {

    @Around("@annotation(Loggable)")
    public Object logExecution(ProceedingJoinPoint joinPoint) throws Throwable {
        String method = joinPoint.getSignature().toShortString();
        log.info("Entering: {}", method);

        long start = System.currentTimeMillis();
        try {
            Object result = joinPoint.proceed();
            log.info("Exiting: {} ({}ms)", method, System.currentTimeMillis() - start);
            return result;
        } catch (Exception e) {
            log.error("Exception in: {}", method, e);
            throw e;
        }
    }
}

// Custom annotation for marking methods
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Loggable {}
```

### Pointcut Expressions

| Expression                                      | Matches                                 |
| ----------------------------------------------- | --------------------------------------- |
| `execution(* com.example.service.*.*(..))`      | All methods in service package          |
| `@annotation(com.example.Loggable)`             | Methods annotated with @Loggable        |
| `@within(org.springframework.stereotype.Service)` | All methods in @Service classes       |
| `bean(userService)`                             | All methods on bean named userService   |

### Proxy Mechanism Rules

- Spring AOP uses **JDK dynamic proxy** (interface-based) or **CGLIB proxy** (class-based)
- **Self-invocation does NOT trigger AOP** — calling `this.method()` bypasses the proxy

```java
@Service
public class UserService {
    @Transactional
    public void createUser(User user) {
        userRepository.save(user);
        this.sendWelcomeEmail(user); // AOP NOT applied — self-invocation
    }

    @Async
    public void sendWelcomeEmail(User user) { ... } // Will NOT run async
}

// Fix: extract to separate bean or use self-injection
@Service
public class UserService {
    private final EmailService emailService; // Separate bean

    @Transactional
    public void createUser(User user) {
        userRepository.save(user);
        emailService.sendWelcomeEmail(user); // AOP applied correctly
    }
}
```

### AOP Rules

- Use `@Around` for timing, logging, retry — use `@Before`/`@After` for simpler cross-cutting
- Never put business logic in aspects — only cross-cutting concerns
- Self-invocation bypasses proxy — extract to separate bean when AOP is needed
- AOP adds runtime overhead — avoid on hot paths with millions of calls/sec

---

## 3. Transaction Management

### @Transactional Behavior

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    // Read-write transaction (default)
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = Order.from(request);
        return orderRepository.save(order);
    }

    // Read-only transaction — enables optimizations
    @Transactional(readOnly = true)
    public Order findById(Long id) {
        return orderRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Order", id));
    }
}
```

### Propagation Levels

| Propagation      | Behavior                                          | Use Case                         |
| ---------------- | ------------------------------------------------- | -------------------------------- |
| `REQUIRED`       | Join existing or create new (default)             | Most business operations         |
| `REQUIRES_NEW`   | Always create new, suspend existing               | Audit logging, independent ops   |
| `SUPPORTS`       | Join existing or run non-transactional             | Read-only queries                |
| `NOT_SUPPORTED`  | Suspend existing, run non-transactional            | Non-critical operations          |
| `MANDATORY`      | Must run in existing transaction, else exception   | Operations requiring caller tx   |
| `NEVER`          | Must NOT run in transaction, else exception        | Operations that must not be tx   |
| `NESTED`         | Nested transaction with savepoint                  | Partial rollback scenarios       |

### Rollback Rules

```java
// Default: rollback on unchecked exceptions (RuntimeException), NOT on checked
@Transactional
public void process() {
    // RuntimeException → rollback
    // IOException (checked) → commit (NOT rolled back)
}

// Explicit rollback for checked exceptions
@Transactional(rollbackFor = IOException.class)
public void processFile() throws IOException { ... }

// No rollback for specific runtime exceptions
@Transactional(noRollbackFor = BusinessValidationException.class)
public void validate() { ... }
```

### Transaction Pitfalls

- **Self-invocation**: `@Transactional` on method B called from method A in the same class does NOT create transaction (proxy bypass)
- **Private methods**: `@Transactional` on private methods is silently ignored
- **Exception swallowed**: Catching exception inside `@Transactional` prevents rollback
- **Long transactions**: Never call external APIs inside `@Transactional` — hold locks too long

```java
// Bad: exception caught inside transaction — no rollback
@Transactional
public void riskyOperation() {
    try {
        repository.save(entity);
        externalApi.call(); // Fails
    } catch (Exception e) {
        log.error("Failed", e); // Transaction commits despite failure
    }
}

// Good: let exception propagate
@Transactional
public void riskyOperation() {
    repository.save(entity);
    // External API call should be outside @Transactional
}
```

---

## 4. Event System

### Publishing Events

```java
// Event class (record preferred for immutability)
public record OrderCreatedEvent(Long orderId, Long userId, BigDecimal amount) {}

// Publishing
@Service
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(Order.from(request));
        eventPublisher.publishEvent(new OrderCreatedEvent(
            order.getId(), order.getUserId(), order.getAmount()
        ));
        return order;
    }
}
```

### Consuming Events

```java
@Component
public class OrderEventHandler {

    // Synchronous listener — runs in publisher's thread and transaction
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        log.info("Order created: {}", event.orderId());
    }

    // Async listener — runs in separate thread
    @Async
    @EventListener
    public void sendOrderNotification(OrderCreatedEvent event) {
        notificationService.send(event.userId(), "Order confirmed");
    }

    // Runs AFTER transaction commits — safe for side effects
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void afterOrderCommitted(OrderCreatedEvent event) {
        externalApi.notifyPartner(event.orderId());
    }
}
```

### Event Rules

| Listener Type                 | Transaction Context              | Use Case                        |
| ----------------------------- | -------------------------------- | ------------------------------- |
| `@EventListener`             | Same transaction as publisher    | In-process sync processing      |
| `@Async @EventListener`      | No transaction (new thread)      | Fire-and-forget side effects    |
| `@TransactionalEventListener` | Runs after tx phase (e.g., commit) | External calls after commit   |

- Use `@TransactionalEventListener(AFTER_COMMIT)` for operations that must not execute if transaction rolls back
- `@TransactionalEventListener` events are NOT delivered if no transaction is active
- For cross-service events, use messaging (Kafka, NATS) instead of `ApplicationEvent`

---

## 5. Bean Lifecycle

### Lifecycle Callbacks

```java
@Component
public class CacheWarmer {

    @PostConstruct
    public void init() {
        // Called after dependency injection is complete
        // Use for initialization logic
        loadCache();
    }

    @PreDestroy
    public void cleanup() {
        // Called before bean destruction
        // Use for cleanup (close connections, flush buffers)
        clearCache();
    }
}
```

### Lifecycle Order

```text
1. Constructor called
2. Dependencies injected
3. @PostConstruct
4. ApplicationContext ready
5. ... (application runs) ...
6. @PreDestroy
7. Bean destroyed
```

### Rules

- Prefer `@PostConstruct` over `InitializingBean.afterPropertiesSet()`
- `@PostConstruct` runs once — do not put retry logic here
- Keep `@PostConstruct` fast — slow initialization delays application startup
- Use `ApplicationRunner` or `CommandLineRunner` for Boot-specific startup tasks

---

## 6. Spring MVC Core

### HandlerInterceptor

```java
@Component
public class RequestTimingInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {
        request.setAttribute("startTime", System.currentTimeMillis());
        return true; // Continue processing
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler, Exception ex) {
        long start = (Long) request.getAttribute("startTime");
        long duration = System.currentTimeMillis() - start;
        log.info("{} {} completed in {}ms",
            request.getMethod(), request.getRequestURI(), duration);
    }
}

// Register interceptor
@Configuration
public class WebConfig implements WebMvcConfigurer {
    private final RequestTimingInterceptor timingInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(timingInterceptor)
            .addPathPatterns("/api/**")
            .excludePathPatterns("/api/health");
    }
}
```

### Filter vs Interceptor vs AOP

| Mechanism      | Level             | Access To                    | Use Case                       |
| -------------- | ----------------- | ---------------------------- | ------------------------------ |
| `Filter`       | Servlet           | Request/Response only        | Authentication, CORS, logging  |
| `Interceptor`  | Spring MVC        | Handler method info          | Request timing, authorization  |
| `AOP`          | Spring bean       | Method args, return value    | Business cross-cutting concerns|

- Filters run before Spring MVC — use for servlet-level concerns
- Interceptors run within Spring MVC — use when handler info is needed
- AOP applies to any Spring bean — use for service/repository concerns

---

## 7. Validation

### Bean Validation with Custom Validators

```java
// Custom constraint annotation
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PhoneNumberValidator.class)
public @interface ValidPhoneNumber {
    String message() default "Invalid phone number format";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Validator implementation
public class PhoneNumberValidator
    implements ConstraintValidator<ValidPhoneNumber, String> {

    private static final Pattern PHONE_PATTERN =
        Pattern.compile("^\\d{2,3}-\\d{3,4}-\\d{4}$");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) return true; // Use @NotNull for null checks
        return PHONE_PATTERN.matcher(value).matches();
    }
}
```

### Validation Groups

```java
// Define groups
public interface OnCreate {}
public interface OnUpdate {}

// Use groups on fields
public record UserRequest(
    @Null(groups = OnCreate.class)
    @NotNull(groups = OnUpdate.class)
    Long id,

    @NotBlank(groups = {OnCreate.class, OnUpdate.class})
    String name
) {}

// Apply group in controller
@PostMapping
public UserResponse create(@Validated(OnCreate.class) @RequestBody UserRequest req) { ... }

@PutMapping("/{id}")
public UserResponse update(@Validated(OnUpdate.class) @RequestBody UserRequest req) { ... }
```

### Validation Rules

- Validate at API boundaries — do not trust input from controllers
- Use `@Validated` (Spring) over `@Valid` (Jakarta) when validation groups are needed
- Custom validators should be stateless and thread-safe
- Return `true` for `null` values — let `@NotNull` handle null checking separately

---

## 8. Task Scheduling

### @Scheduled

```java
@Configuration
@EnableScheduling
public class SchedulingConfig {}

@Component
public class CleanupTask {

    @Scheduled(fixedDelay = 60_000) // 60s after previous completion
    public void cleanExpiredSessions() {
        sessionRepository.deleteExpired();
    }

    @Scheduled(cron = "0 0 2 * * *") // Daily at 2:00 AM
    public void generateDailyReport() {
        reportService.generateDaily();
    }

    @Scheduled(fixedRate = 30_000) // Every 30s regardless of previous
    public void refreshCache() {
        cacheService.refresh();
    }
}
```

### @Async

```java
@Configuration
@EnableAsync
public class AsyncConfig {}

@Service
public class NotificationService {

    @Async
    public CompletableFuture<Void> sendEmail(String to, String body) {
        emailClient.send(to, body);
        return CompletableFuture.completedFuture(null);
    }
}
```

### Scheduling Rules

| Parameter    | Behavior                                    |
| ------------ | ------------------------------------------- |
| `fixedDelay` | Wait N ms after previous execution finishes |
| `fixedRate`  | Execute every N ms (may overlap if slow)    |
| `cron`       | Cron expression for calendar-based schedule |

- `@Scheduled` methods must return `void` and take no parameters
- `@Async` methods must return `void` or `CompletableFuture`
- `@Async` on self-invoked methods does NOT work (proxy bypass) — same as `@Transactional`
- Default executor is single-threaded — configure `TaskScheduler` for parallel scheduled tasks

---

## 9. Anti-Patterns

- Field injection with `@Autowired` — use constructor injection
- Storing mutable state in singleton beans — causes concurrency bugs
- Self-invocation expecting AOP/proxy behavior (`@Transactional`, `@Async`, `@Cacheable`)
- Calling external APIs inside `@Transactional` — holds DB locks
- Catching exceptions inside `@Transactional` — prevents rollback
- `@Transactional` on private methods — silently ignored
- Using `ApplicationEvent` for cross-service communication — use messaging
- Heavy initialization in `@PostConstruct` — delays startup
- Ignoring `@TransactionalEventListener` phase — side effects may execute before commit
- Using `@Scheduled(fixedRate)` for long-running tasks without overlap protection
