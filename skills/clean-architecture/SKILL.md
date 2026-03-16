# Clean Architecture / Hexagonal Architecture Rules

## 1. Architecture Principles

### Dependency Rule

- Dependencies always point inward: outer layers depend on inner layers, never the reverse
- The domain layer is the center and has zero external dependencies
- Framework, database, and UI are implementation details that belong to the outermost layers
- Changes in infrastructure must never require changes in domain logic

### Separation of Concerns

| Layer          | Responsibility                          | Changes When                  |
| -------------- | --------------------------------------- | ----------------------------- |
| Domain         | Business rules, entities, value objects | Business requirements change  |
| Application    | Use case orchestration, ports           | Workflow or use case changes  |
| Infrastructure | Database, messaging, external APIs      | Technology or vendor changes  |
| Presentation   | HTTP, CLI, event listener adapters      | Interface or protocol changes |

### Core Principles

- Business rules are independent of frameworks, databases, and delivery mechanisms
- Each layer has a well-defined boundary with explicit contracts (interfaces)
- Inner layers define interfaces (ports) that outer layers implement (adapters)
- The architecture makes the system testable without external dependencies

---

## 2. Layer Structure

### Layer Hierarchy (Inside-Out)

```text
┌─────────────────────────────────────────────┐
│              Presentation Layer              │  Controllers, CLI, Event Listeners
├─────────────────────────────────────────────┤
│             Infrastructure Layer             │  DB, External API, Messaging
├─────────────────────────────────────────────┤
│              Application Layer               │  Use Cases, Application Services
├─────────────────────────────────────────────┤
│                Domain Layer                  │  Entities, Value Objects, Domain Services
└─────────────────────────────────────────────┘
         ▲ Dependencies point inward ▲
```

### Layer Responsibilities

#### Domain Layer (Innermost)

- Entities with identity and lifecycle
- Value objects (immutable, equality by value)
- Domain services (cross-aggregate logic)
- Domain events
- Repository interfaces (outbound ports)
- No framework annotations, no infrastructure imports

#### Application Layer

- Use case classes / application services
- Inbound port interfaces (what the system can do)
- Outbound port interfaces (what the system needs)
- Command and query objects
- Transaction boundary management
- Event publishing orchestration

#### Infrastructure Layer

- Repository implementations (JPA, R2DBC, Exposed)
- External API clients
- Message broker producers/consumers
- File system access
- Cache implementations
- Framework-specific configuration

#### Presentation Layer

- REST controllers / GraphQL resolvers
- Request/response DTOs
- Input validation (format-level, not business-level)
- Authentication filter integration
- API documentation annotations

---

## 3. Ports and Adapters Pattern

### Inbound Ports (Driving Side)

Inbound ports define what the application can do. They are interfaces that the application layer exposes and the presentation layer invokes.

```kotlin
// Inbound port — defines a use case
interface CreateOrderUseCase {
    fun execute(command: CreateOrderCommand): OrderId
}

interface GetOrderQuery {
    fun execute(orderId: OrderId): OrderDetailResult
}
```

### Outbound Ports (Driven Side)

Outbound ports define what the application needs from the outside world. They are interfaces defined in the application or domain layer and implemented in the infrastructure layer.

```kotlin
// Outbound port — defined in domain layer
interface OrderRepository {
    fun findById(id: OrderId): Order?
    fun save(order: Order)
    fun delete(id: OrderId)
}

// Outbound port — defined in application layer
interface PaymentGateway {
    fun charge(orderId: OrderId, amount: Money): PaymentResult
}

// Outbound port — defined in application layer
interface NotificationSender {
    fun sendOrderConfirmation(customerId: CustomerId, orderId: OrderId)
}
```

### Inbound Adapters

Inbound adapters translate external requests into use case invocations.

```kotlin
// REST adapter (inbound)
@RestController
@RequestMapping("/api/v1/orders")
class OrderController(
    private val createOrderUseCase: CreateOrderUseCase,
    private val getOrderQuery: GetOrderQuery
) {
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun createOrder(@Valid @RequestBody request: CreateOrderRequest): CreateOrderResponse {
        val command = request.toCommand()
        val orderId = createOrderUseCase.execute(command)
        return CreateOrderResponse(orderId.value)
    }

    @GetMapping("/{id}")
    fun getOrder(@PathVariable id: Long): OrderDetailResponse {
        val result = getOrderQuery.execute(OrderId(id))
        return result.toResponse()
    }
}

// Message listener adapter (inbound)
@Component
class PaymentEventListener(
    private val handlePaymentResultUseCase: HandlePaymentResultUseCase
) {
    @KafkaListener(topics = ["payment-results"])
    fun onPaymentResult(event: PaymentResultEvent) {
        val command = event.toCommand()
        handlePaymentResultUseCase.execute(command)
    }
}
```

### Outbound Adapters

Outbound adapters implement outbound ports with specific technology.

```kotlin
// JPA adapter (outbound)
@Repository
class JpaOrderRepository(
    private val jpaRepository: OrderJpaRepository,
    private val mapper: OrderEntityMapper
) : OrderRepository {

    override fun findById(id: OrderId): Order? {
        return jpaRepository.findByIdOrNull(id.value)?.let { mapper.toDomain(it) }
    }

    override fun save(order: Order) {
        val entity = mapper.toEntity(order)
        jpaRepository.save(entity)
    }

    override fun delete(id: OrderId) {
        jpaRepository.deleteById(id.value)
    }
}

// External API adapter (outbound)
@Component
class StripePaymentGateway(
    private val restClient: RestClient
) : PaymentGateway {

    override fun charge(orderId: OrderId, amount: Money): PaymentResult {
        val response = restClient.post()
            .uri("/charges")
            .body(StripeChargeRequest(orderId.value, amount.value))
            .retrieve()
            .body(StripeChargeResponse::class.java)
            ?: throw PaymentGatewayException("Empty response from payment gateway")

        return response.toDomain()
    }
}
```

### Port and Adapter Summary

| Type             | Defined In           | Implemented In | Example              |
| ---------------- | -------------------- | -------------- | -------------------- |
| Inbound port     | Application          | Application    | `CreateOrderUseCase` |
| Inbound adapter  | Presentation         | Presentation   | `OrderController`    |
| Outbound port    | Domain / Application | Infrastructure | `OrderRepository`    |
| Outbound adapter | Infrastructure       | Infrastructure | `JpaOrderRepository` |

---

## 4. Package Structure

### Recommended Layout (Kotlin + Spring Boot)

```text
com.example.order/
├── domain/                          # Domain layer
│   ├── model/
│   │   ├── Order.kt                # Aggregate root
│   │   ├── OrderLine.kt            # Entity within aggregate
│   │   ├── OrderId.kt              # Value object (ID)
│   │   ├── OrderStatus.kt          # Enum
│   │   └── Money.kt                # Value object
│   ├── event/
│   │   ├── DomainEvent.kt          # Event marker interface
│   │   └── OrderConfirmedEvent.kt  # Domain event
│   ├── service/
│   │   └── OrderPricingService.kt  # Domain service
│   └── repository/
│       └── OrderRepository.kt      # Outbound port (interface)
│
├── application/                     # Application layer
│   ├── port/
│   │   ├── inbound/
│   │   │   ├── CreateOrderUseCase.kt
│   │   │   └── GetOrderQuery.kt
│   │   └── outbound/
│   │       ├── PaymentGateway.kt
│   │       └── NotificationSender.kt
│   ├── service/
│   │   ├── CreateOrderService.kt   # Use case implementation
│   │   └── OrderQueryService.kt    # Query implementation
│   └── dto/
│       ├── CreateOrderCommand.kt   # Input command
│       └── OrderDetailResult.kt    # Output result
│
├── infrastructure/                  # Infrastructure layer
│   ├── persistence/
│   │   ├── entity/
│   │   │   └── OrderJpaEntity.kt   # JPA entity
│   │   ├── repository/
│   │   │   ├── OrderJpaRepository.kt   # Spring Data interface
│   │   │   └── JpaOrderRepository.kt   # Outbound adapter
│   │   └── mapper/
│   │       └── OrderEntityMapper.kt    # Entity ↔ Domain mapper
│   ├── external/
│   │   └── StripePaymentGateway.kt # External API adapter
│   ├── messaging/
│   │   └── KafkaNotificationSender.kt # Messaging adapter
│   └── config/
│       └── PersistenceConfig.kt    # Infrastructure config
│
└── presentation/                    # Presentation layer
    ├── controller/
    │   └── OrderController.kt      # REST inbound adapter
    ├── dto/
    │   ├── CreateOrderRequest.kt   # API request DTO
    │   └── OrderDetailResponse.kt  # API response DTO
    └── mapper/
        └── OrderResponseMapper.kt  # Request/Response ↔ Command/Result
```

### Package Dependency Rules

```text
presentation  → application  (invokes use cases)
infrastructure → domain      (implements repository ports)
infrastructure → application (implements outbound ports)
application   → domain       (uses domain model)
domain        → (nothing)    (no outward dependencies)
```

- `domain` package must not import from `application`, `infrastructure`, or `presentation`
- `application` package must not import from `infrastructure` or `presentation`
- `presentation` must not import from `infrastructure` directly
- Cross-cutting via dependency injection only (Spring wires adapters to ports)

---

## 5. Data Transformation Between Layers

### Transformation Flow

```text
Request DTO → Command → Domain Entity → JPA Entity → Database
Database → JPA Entity → Domain Entity → Result DTO → Response DTO
```

### Layer-Specific Data Objects

| Layer          | Data Object           | Purpose               | Framework Annotations |
| -------------- | --------------------- | --------------------- | --------------------- |
| Presentation   | Request / Response    | API contract          | `@Valid`, Jackson     |
| Application    | Command / Result      | Use case input/output | None                  |
| Domain         | Entity / Value Object | Business model        | None                  |
| Infrastructure | JPA Entity / Row      | Persistence mapping   | `@Entity`, `@Table`   |

### Mapping Examples

```kotlin
// Presentation → Application
fun CreateOrderRequest.toCommand() = CreateOrderCommand(
    customerId = CustomerId(this.customerId),
    productId = ProductId(this.productId),
    quantity = this.quantity
)

// Application → Presentation
fun OrderDetailResult.toResponse() = OrderDetailResponse(
    id = this.orderId.value,
    status = this.status.name,
    totalAmount = this.totalAmount.value,
    createdAt = this.createdAt
)

// Infrastructure: JPA Entity ↔ Domain
@Component
class OrderEntityMapper {
    fun toDomain(entity: OrderJpaEntity): Order {
        return Order(
            id = OrderId(entity.id),
            customerId = CustomerId(entity.customerId),
            status = entity.status,
            lines = entity.lines.map { toOrderLine(it) }
        )
    }

    fun toEntity(domain: Order): OrderJpaEntity {
        return OrderJpaEntity(
            id = domain.id.value,
            customerId = domain.customerId.value,
            status = domain.status
        ).also { entity ->
            entity.lines = domain.lines.map { toLineEntity(it, entity) }
        }
    }
}
```

### Mapping Rules

- Each layer boundary has its own data objects -- never pass JPA entities to controllers
- Mapping logic lives at the boundary of the outer layer (adapter side)
- Domain objects never depend on DTO or JPA entity classes
- Use extension functions or dedicated mapper classes for conversions
- Avoid deep nested mapping in a single function -- compose small mapping functions

---

## 6. Dependency Inversion Principle (DIP)

### Core Mechanism

The domain and application layers define interfaces (ports) that the infrastructure layer implements. Spring's dependency injection wires the concrete implementations at runtime.

```kotlin
// Domain layer defines the interface
interface OrderRepository {
    fun findById(id: OrderId): Order?
    fun save(order: Order)
}

// Infrastructure layer implements it
@Repository
class JpaOrderRepository(
    private val jpaRepository: OrderJpaRepository
) : OrderRepository {
    override fun findById(id: OrderId): Order? = ...
    override fun save(order: Order) = ...
}

// Application layer depends only on the interface
@Service
class CreateOrderService(
    private val orderRepository: OrderRepository,  // Port, not adapter
    private val paymentGateway: PaymentGateway      // Port, not adapter
) : CreateOrderUseCase {
    override fun execute(command: CreateOrderCommand): OrderId { ... }
}
```

### DIP Benefits

| Without DIP                                | With DIP                                    |
| ------------------------------------------ | ------------------------------------------- |
| Service depends on `JpaOrderRepository`    | Service depends on `OrderRepository` (port) |
| Changing DB requires changing service code | Changing DB only requires new adapter       |
| Testing requires real DB or mock framework | Testing uses simple fake implementation     |
| Domain coupled to framework                | Domain is framework-free                    |

### DIP Application Rules

- Define interfaces in the layer that needs the capability (domain or application)
- Implement interfaces in the outer layer that provides the capability (infrastructure)
- Never create an interface just to have an interface -- use DIP only when the boundary is meaningful
- Spring `@Service`, `@Repository`, `@Component` annotations belong on implementations, not on ports

---

## 7. Use Case / Application Service Pattern

### Use Case Interface (Inbound Port)

```kotlin
// One interface per use case — clear, focused, independently testable
interface CreateOrderUseCase {
    fun execute(command: CreateOrderCommand): OrderId
}

interface CancelOrderUseCase {
    fun execute(command: CancelOrderCommand)
}

interface GetOrderDetailQuery {
    fun execute(orderId: OrderId): OrderDetailResult
}
```

### Use Case Implementation

```kotlin
@Service
class CreateOrderService(
    private val orderRepository: OrderRepository,
    private val productRepository: ProductRepository,
    private val eventPublisher: ApplicationEventPublisher
) : CreateOrderUseCase {

    @Transactional
    override fun execute(command: CreateOrderCommand): OrderId {
        // 1. Load domain objects
        val product = productRepository.findById(command.productId)
            ?: throw EntityNotFoundException("Product", command.productId)

        // 2. Execute domain logic (delegate to domain)
        val order = Order.create(
            customerId = command.customerId,
            product = product,
            quantity = command.quantity
        )

        // 3. Persist
        orderRepository.save(order)

        // 4. Publish domain events
        order.pullEvents().forEach { eventPublisher.publishEvent(it) }

        // 5. Return result
        return order.id
    }
}
```

### Use Case Design Rules

- One class per use case (Single Responsibility)
- Use cases are thin orchestrators -- business logic belongs in domain objects
- Use cases handle transaction boundaries, not domain objects
- Input is a command/query object, output is a result object or domain ID
- Never return domain entities from use cases -- return result DTOs or IDs
- Use case names describe business actions, not technical operations

### Command vs Query Separation (CQS)

| Aspect  | Command              | Query                             |
| ------- | -------------------- | --------------------------------- |
| Purpose | Change state         | Read state                        |
| Return  | Void or created ID   | Result DTO                        |
| Side    | Write side           | Read side                         |
| Tx      | `@Transactional`     | `@Transactional(readOnly = true)` |
| Example | `CreateOrderUseCase` | `GetOrderDetailQuery`             |

---

## 8. Testability by Design

### Testing Strategy Per Layer

| Layer          | Test Type        | Dependencies             | Speed  |
| -------------- | ---------------- | ------------------------ | ------ |
| Domain         | Unit test        | None (pure logic)        | Fast   |
| Application    | Unit test        | Fake ports (in-memory)   | Fast   |
| Infrastructure | Integration test | Testcontainers, WireMock | Slow   |
| Presentation   | Slice test       | MockMvc, WebTestClient   | Medium |

### Domain Layer Test (No Dependencies)

```kotlin
class OrderTest {
    @Test
    fun `should add line to order`() {
        val order = Order.create(customerId, product, quantity = 2)

        assertThat(order.totalAmount()).isEqualTo(Money.of(2000))
    }

    @Test
    fun `should reject more than 10 items`() {
        val order = Order.create(customerId, product, quantity = 1)
        repeat(9) { order.addLine(product, 1) }

        assertThrows<IllegalArgumentException> {
            order.addLine(product, 1)
        }
    }
}
```

### Application Layer Test (Fake Ports)

```kotlin
class CreateOrderServiceTest {
    private val orderRepository = FakeOrderRepository()
    private val productRepository = FakeProductRepository()
    private val eventPublisher = FakeEventPublisher()

    private val sut = CreateOrderService(orderRepository, productRepository, eventPublisher)

    @Test
    fun `should create order and publish event`() {
        productRepository.save(product)

        val orderId = sut.execute(CreateOrderCommand(customerId, productId, quantity = 2))

        val saved = orderRepository.findById(orderId)
        assertThat(saved).isNotNull
        assertThat(eventPublisher.publishedEvents).hasSize(1)
    }
}

// Fake implementation for testing
class FakeOrderRepository : OrderRepository {
    private val store = mutableMapOf<OrderId, Order>()

    override fun findById(id: OrderId): Order? = store[id]
    override fun save(order: Order) { store[order.id] = order }
    override fun delete(id: OrderId) { store.remove(id) }
}
```

### Testability Rules

- Domain layer tests require zero mocking -- if mocking is needed, the domain has external dependencies (violation)
- Application layer tests use fake implementations of ports, not mocks
- Infrastructure tests verify that adapters correctly translate between domain and technology
- Presentation tests verify HTTP contract (status codes, JSON structure), not business logic
- If a class is hard to test, it likely violates separation of concerns -- fix the design, not the test

---

## 9. Anti-Patterns

### Domain Layer Violations

- **Framework annotations in domain**: `@Entity`, `@Component`, `@Transactional` on domain classes couples domain to framework
- **Infrastructure imports in domain**: Domain classes importing JPA, Spring, or HTTP client packages
- **Anemic domain model**: Domain objects with only getters/setters and all logic in services
- **Domain returning infrastructure types**: Domain methods returning `Page<T>`, `ResponseEntity`, or JPA entities

### Dependency Violations

- **Bidirectional dependencies**: Application layer depending on infrastructure and infrastructure depending back on application
- **Skipping layers**: Controller directly calling repository without going through use case
- **Shared mutable state**: Passing JPA entities across layer boundaries (lazy loading failures, unintended mutations)

### Structural Violations

- **God use case**: Single application service handling dozens of unrelated operations
- **Leaky abstraction**: Outbound port method signatures exposing infrastructure details (e.g., `fun findByJpql(query: String)`)
- **DTO explosion**: Creating separate DTOs for every minor variation instead of reusing where appropriate
- **Premature abstraction**: Creating ports and adapters for internal modules that will never have multiple implementations

### Common Mistakes

| Mistake                              | Why It Hurts                                | Fix                                                     |
| ------------------------------------ | ------------------------------------------- | ------------------------------------------------------- |
| JPA entity as domain entity          | Domain coupled to persistence framework     | Separate domain model and JPA entity                    |
| Business logic in controller         | Untestable without HTTP context             | Move to domain or application layer                     |
| Repository returning DTOs            | Mixes persistence and presentation concerns | Return domain objects, map at boundary                  |
| `@Transactional` on domain service   | Domain depends on Spring                    | Put `@Transactional` on application service             |
| Using Spring events as domain events | Domain coupled to Spring event system       | Domain defines events, application publishes via Spring |

---

## 10. Related Rules

| Rule File                  | When to Reference                                             |
| -------------------------- | ------------------------------------------------------------- |
| `ddd.md`                   | Designing entities, aggregates, value objects, domain events  |
| `code-quality.md`          | Abstraction layers, modularity, single responsibility         |
| `spring-framework.md`      | IoC, DI, `@Transactional`, event publishing                   |
| `spring-jpa.md`            | Repository implementation, N+1 prevention, entity conventions |
| `kotlin-convention.md`     | Sealed classes, extension functions, naming conventions       |
| `testing-unit.md`              | Writing tests for use cases and domain logic                  |
| `error-handling.md`        | Exception hierarchy, business vs system exceptions            |
| `spring-error-handling.md` | `@ControllerAdvice`, `ErrorCode` enum, error response format  |

---

## Additional Resources

- Alistair Cockburn, "Hexagonal Architecture" (original article, 2005)
- Robert C. Martin, "Clean Architecture" concepts and dependency rule
- Vaughn Vernon, "Implementing Domain-Driven Design" (architecture patterns chapter)
- Netflix Tech Blog, "Ready for changes with Hexagonal Architecture"
- Herberto Graca, "DDD, Hexagonal, Onion, Clean, CQRS, How I put it all together" (blog series)
- Tom Hombergs, "Get Your Hands Dirty on Clean Architecture" (Spring Boot examples)
- Baeldung, "Hexagonal Architecture in Java" (tutorial series)
- JetBrains, "Kotlin + Spring Boot project structure" (official guide)
