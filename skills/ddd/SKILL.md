# Domain-Driven Design (DDD) Skill

Domain-Driven Design patterns and practices for building complex software systems. Use when designing domain models, implementing business logic, or structuring domain-centric applications.

## Core Philosophy

- Focus on domain knowledge and business logic
- Bridge the gap between domain experts and code
- Create a Ubiquitous Language shared by all stakeholders
- Iteratively refine the model through collaboration

---

## 1. Value Objects

Objects defined by their attributes, not identity. Immutable and interchangeable.

### Value Object Properties

- **Immutability**: Value cannot change after creation
- **Interchangeability**: Equal values can replace each other
- **Equality by Value**: Two objects are equal if all attributes match
- **No Side Effects**: Methods don't modify state

### Value Object Example

```kotlin
// Value Object example
data class Money(val amount: BigDecimal, val currency: Currency) {
    init {
        require(amount >= BigDecimal.ZERO) { "Amount must be non-negative" }
    }

    fun add(other: Money): Money {
        require(currency == other.currency) { "Currency mismatch" }
        return Money(amount + other.amount, currency)
    }
}

data class Email(val value: String) {
    init {
        require(value.contains("@")) { "Invalid email format" }
    }
}
```

### Value Object Decision Criteria

- No identity needed (two $10 bills are identical)
- Measured/compared by attributes
- Immutable behavior desired

---

## 2. Entities

Objects with distinct identity that persists through state changes.

### Entity Properties

- **Identity**: Unique identifier distinguishes entities
- **Mutability**: State can change over lifecycle
- **Continuity**: Same identity across state changes
- **Equality by Identity**: Two entities equal if IDs match

### Entity Example

```kotlin
class User(
    val id: UserId,
    private var name: UserName,
    private var email: Email
) {
    fun changeName(newName: UserName) {
        this.name = newName // Mutable state
    }

    fun changeEmail(newEmail: Email) {
        validateEmailChange(newEmail)
        this.email = newEmail
    }

    override fun equals(other: Any?) = other is User && other.id == id
    override fun hashCode() = id.hashCode()
}
```

### Entity Decision Criteria

- Lifecycle continuity matters
- Tracking identity is essential
- State changes over time

---

## 3. Domain Services

Operations that don't naturally belong to Entity or Value Object.

### When to Use Domain Services

- Operation involves multiple aggregates
- No natural home in existing objects
- Represents a domain concept/activity

### Domain Service Example

```kotlin
// Domain Service example
class TransferService(
    private val accountRepository: AccountRepository
) {
    fun transfer(fromAccountId: AccountId, toAccountId: AccountId, amount: Money) {
        val fromAccount = accountRepository.findById(fromAccountId)
        val toAccount = accountRepository.findById(toAccountId)

        fromAccount.withdraw(amount)
        toAccount.deposit(amount)

        accountRepository.save(fromAccount)
        accountRepository.save(toAccount)
    }
}

// Service interface in domain layer
interface UserRepository {
    fun findById(id: UserId): User?
    fun save(user: User)
    fun exists(email: Email): Boolean
}
```

### Domain Service Anti-Patterns

- Anemic domain model (all logic in services)
- Services as dumping ground for procedures
- Business rules leaking to application layer

---

## 4. Repositories

Abstract collection-like interface for persisting and retrieving aggregates.

### Repository Responsibilities

- Retrieve aggregates by identity
- Store aggregates
- Delete aggregates
- Query by specification (optional)

### Repository Example

```kotlin
// Repository interface (domain layer)
interface OrderRepository {
    fun findById(id: OrderId): Order?
    fun save(order: Order)
    fun delete(order: Order)
    fun findByCustomer(customerId: CustomerId): List<Order>
}

// Implementation (infrastructure layer)
class JpaOrderRepository(
    private val entityManager: EntityManager
) : OrderRepository {
    override fun findById(id: OrderId): Order? {
        return entityManager.find(Order::class.java, id)
    }

    override fun save(order: Order) {
        entityManager.persist(order)
    }
}
```

### Repository Guidelines

- One repository per aggregate
- Work with aggregate roots only
- Hide persistence details
- Return domain objects, not data structures

---

## 5. Aggregates

Cluster of domain objects treated as a single unit for data changes.

### Aggregate Rules

- **Single Root**: Only aggregate root has global identity
- **Boundary**: Define what's inside vs outside
- **Invariant Protection**: Root ensures consistency within boundary
- **Reference by Identity**: External references only to root

### Aggregate Example

```kotlin
// Aggregate Root
class Order(
    val id: OrderId,
    private val customerId: CustomerId,
    private var status: OrderStatus,
    private val lines: MutableList<OrderLine> = mutableListOf()
) {
    // Business rule: max 10 items per order
    fun addLine(product: Product, quantity: Int) {
        require(lines.size < 10) { "Maximum 10 items per order" }
        require(quantity > 0) { "Quantity must be positive" }
        lines.add(OrderLine(product.id, product.price, quantity))
    }

    fun totalAmount(): Money {
        return lines.fold(Money.ZERO) { acc, line ->
            acc + line.subtotal()
        }
    }

    fun confirm() {
        require(lines.isNotEmpty()) { "Cannot confirm empty order" }
        status = OrderStatus.CONFIRMED
    }

    internal fun isConfirmed() = status == OrderStatus.CONFIRMED
}

// Entity within aggregate (no repository)
class OrderLine(
    val productId: ProductId,
    val unitPrice: Money,
    val quantity: Int
) {
    fun subtotal() = unitPrice.multiply(quantity)
}
```

### Aggregate Design Rules

1. Keep aggregates small
2. Reference other aggregates by ID only
3. Use eventual consistency across aggregates
4. Root controls all access to internals

---

## 6. Factories

Encapsulate complex object creation logic.

### When to Use Factories

- Creation logic is complex
- Multiple steps required
- Validation during creation
- Hiding concrete types

### Factory Example

```kotlin
// Factory method on aggregate
class Order {
    companion object {
        fun create(
            customerId: CustomerId,
            product: Product,
            quantity: Int
        ): Order {
            require(quantity > 0) { "Quantity must be positive" }
            val order = Order(OrderId.generate(), customerId, OrderStatus.DRAFT)
            order.addLine(product, quantity)
            return order
        }
    }
}

// Separate factory class
interface OrderFactory {
    fun createFromCart(cart: Cart): Order
}

class DefaultOrderFactory : OrderFactory {
    override fun createFromCart(cart: Cart): Order {
        val order = Order(OrderId.generate(), cart.customerId, OrderStatus.DRAFT)
        cart.items.forEach { order.addLine(it.product, it.quantity) }
        return order
    }
}
```

---

## 7. Specifications

Encapsulate business rules for object selection/validation.

### Specification Use Cases

- Validation: Check if object satisfies rules
- Selection: Query objects matching criteria
- Construction: Build objects to specification

### Specification Example

```kotlin
interface Specification<T> {
    fun isSatisfiedBy(candidate: T): Boolean
    fun and(other: Specification<T>): Specification<T> = AndSpecification(this, other)
    fun or(other: Specification<T>): Specification<T> = OrSpecification(this, other)
}

class ActiveCustomerSpecification : Specification<Customer> {
    override fun isSatisfiedBy(candidate: Customer): Boolean {
        return candidate.isActive &&
               candidate.lastOrderDate.isAfter(LocalDate.now().minusMonths(6))
    }
}

class PremiumCustomerSpecification : Specification<Customer> {
    override fun isSatisfiedBy(candidate: Customer): Boolean {
        return candidate.totalSpent >= Money.of(1000)
    }
}

// Usage
val specification = ActiveCustomerSpecification()
    .and(PremiumCustomerSpecification())

val eligibleCustomers = customers.filter { specification.isSatisfiedBy(it) }
```

---

## 8. Domain Events

Capture things that happened in the domain.

### Domain Event Example

```kotlin
// Event definition
data class OrderConfirmed(
    val orderId: OrderId,
    val customerId: CustomerId,
    val confirmedAt: Instant
) : DomainEvent

// Publishing events
class Order {
    private val events = mutableListOf<DomainEvent>()

    fun confirm() {
        require(lines.isNotEmpty()) { "Cannot confirm empty order" }
        status = OrderStatus.CONFIRMED
        events.add(OrderConfirmed(id, customerId, Instant.now()))
    }

    fun pullEvents(): List<DomainEvent> {
        val result = events.toList()
        events.clear()
        return result
    }
}

// Event handler
class SendConfirmationEmailHandler(
    private val emailService: EmailService
) {
    fun handle(event: OrderConfirmed) {
        emailService.sendConfirmation(event.customerId, event.orderId)
    }
}
```

---

## 9. Application Services

Orchestrate use cases by coordinating domain objects.

### Application Service Responsibilities

- Handle use case requests
- Coordinate domain objects
- Manage transactions
- Handle cross-cutting concerns

### Application Service Example

```kotlin
class OrderApplicationService(
    private val orderRepository: OrderRepository,
    private val productRepository: ProductRepository,
    private val eventPublisher: EventPublisher
) {
    @Transactional
    fun createOrder(command: CreateOrderCommand): OrderId {
        val product = productRepository.findById(command.productId)
            ?: throw ProductNotFoundException(command.productId)

        val order = Order.create(
            customerId = command.customerId,
            product = product,
            quantity = command.quantity
        )

        orderRepository.save(order)

        order.pullEvents().forEach { eventPublisher.publish(it) }

        return order.id
    }

    @Transactional
    fun confirmOrder(command: ConfirmOrderCommand) {
        val order = orderRepository.findById(command.orderId)
            ?: throw OrderNotFoundException(command.orderId)

        order.confirm()
        orderRepository.save(order)

        order.pullEvents().forEach { eventPublisher.publish(it) }
    }
}
```

### Application Service Guidelines

- Keep thin - delegate to domain objects
- Don't put business logic here
- Handle transactions and security
- Transform DTOs to/from domain objects

---

## 10. Architecture Patterns

### Layered Architecture

```text
┌─────────────────────────┐
│   Presentation Layer    │  ← Controllers, Views
├─────────────────────────┤
│   Application Layer     │  ← Application Services
├─────────────────────────┤
│     Domain Layer        │  ← Entities, Value Objects, Domain Services
├─────────────────────────┤
│ Infrastructure Layer    │  ← Repositories, External Services
└─────────────────────────┘
```

### Hexagonal Architecture (Ports & Adapters)

```text
         ┌──────────────────┐
         │   Domain Core    │
         │  (Entities, VOs) │
         └────────┬─────────┘
                  │
     ┌─────────────┼─────────────┐
     │             │             │
 ┌───▼───┐     ┌───▼───┐     ┌───▼───┐
 │Ports  │     │Ports  │     │Ports  │
 │(APIs) │     │(SPIs) │     │(Events)│
 └───┬───┘     └───┬───┘     └───┬───┘
     │             │             │
 ┌───▼───┐     ┌───▼───┐     ┌───▼───┐
 │REST   │     │JPA    │     │Kafka  │
 │Adapter│     │Adapter│     │Adapter│
 └───────┘     └───────┘     └───────┘
```

### Dependency Rule

- Domain layer has no dependencies
- Application layer depends on domain
- Infrastructure depends on domain (via interfaces)

---

## Bounded Contexts

Define boundaries where a particular model applies.

### Context Map Patterns

| Pattern                | Description                                  |
|------------------------|----------------------------------------------|
| ---------------------- | -------------------------------------------- |
| Shared Kernel          | Common model subset shared between contexts  |
| Customer/Supplier      | Upstream/downstream relationship             |
| Conformist             | Accept upstream model as-is                  |
| Anticorruption Layer   | Translate between contexts                   |
| Open Host Service      | Published API for integration                |

### Bounded Context Example

```kotlin
// Bounded Context: Order
package com.company.order.domain

class Order { /* Order context model */ }

// Bounded Context: Inventory
package com.company.inventory.domain

class Order { /* Inventory's view of order */ }

// Anticorruption Layer
class OrderTranslator {
    fun toInventoryOrder(order: com.company.order.domain.Order): Order {
        return Order(
            orderId = order.id.value,
            items = order.lines.map { Item(it.productId, it.quantity) }
        )
    }
}
```

---

## Ubiquitous Language

Shared vocabulary between developers and domain experts.

### Ubiquitous Language Examples

| Domain Term   | Code Element                 |
|---------------|------------------------------|
| ------------- | --------------------------   |
| Order         | `Order` entity               |
| Place Order   | `Order.place()` method       |
| Customer      | `Customer` entity            |
| Confirm       | `Order.confirm()` method     |
| Overdue       | `Invoice.isOverdue()` method |

### Ubiquitous Language Guidelines

- Use same terms in code and conversation
- Rename code when domain terms change
- Document terms in glossary
- Avoid technical jargon in domain discussions

---

## Related Skills

- **clean-architecture**: Architectural patterns complementing DDD
- **spring-jpa**: Repository implementation patterns
- **testing-integration**: Testing domain logic

---

## References

- Domain-Driven Design by Eric Evans
- Implementing Domain-Driven Design by Vaughn Vernon
- Domain-Driven Design Distilled by Vaughn Vernon
- 도메인주도 설계 철저 입문 by 나루세 마사노부 (Japanese book, Korean translation)
