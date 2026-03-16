# Object-Oriented Design Principles (SOLID Advanced)

## 1. SRP (Single Responsibility Principle)

### SRP Core Concept

- A class should have exactly **one reason to change**
- "Reason to change" means one actor or stakeholder whose requirements drive modifications
- SRP is about **people** — separate code that different stakeholders depend on

### Violation Indicators

| Indicator                                       | Description                          |
| ----------------------------------------------- | ------------------------------------ |
| Class name includes "And" or "Manager"          | Multiple responsibilities bundled    |
| Class changes for unrelated feature requests    | Different stakeholders drive changes |
| Many import statements from different domains   | Cross-cutting concerns mixed         |
| Test class requires mocking 5+ dependencies     | Too many collaborators               |
| Methods cluster into groups with no interaction | Separate responsibilities coexist    |

### SRP Practical Application

```kotlin
// Bad: report generation + formatting + delivery in one class
class ReportService(
    private val repository: OrderRepository,
    private val templateEngine: TemplateEngine,
    private val emailSender: EmailSender
) {
    fun generateMonthlyReport(month: YearMonth): Report { ... }
    fun renderAsHtml(report: Report): String { ... }
    fun sendToStakeholders(html: String) { ... }
}

// Good: each responsibility separated
class ReportGenerator(private val repository: OrderRepository) {
    fun generate(month: YearMonth): Report { ... }
}

class ReportRenderer(private val templateEngine: TemplateEngine) {
    fun renderAsHtml(report: Report): String { ... }
}

class ReportDelivery(private val emailSender: EmailSender) {
    fun sendToStakeholders(html: String) { ... }
}
```

### SRP Rules

- If you cannot describe a class's purpose without using "and", split it
- Prefer multiple small classes over one large class
- SRP does not mean one method per class — it means one cohesive responsibility
- Apply SRP at method, class, and module level consistently

---

## 2. OCP (Open-Closed Principle)

### OCP Core Concept

- Software entities should be **open for extension** but **closed for modification**
- Add new behavior by writing new code, not by changing existing code
- Achieved through abstraction: depend on stable interfaces, vary implementations

### Strategy Pattern Application

```kotlin
// Closed for modification: DiscountCalculator never changes
// Open for extension: add new DiscountPolicy implementations
fun interface DiscountPolicy {
    fun calculate(order: Order): Money
}

class PercentageDiscount(private val rate: Double) : DiscountPolicy {
    override fun calculate(order: Order): Money =
        order.totalAmount * rate
}

class FixedAmountDiscount(private val amount: Money) : DiscountPolicy {
    override fun calculate(order: Order): Money = amount
}

// New discount type added without modifying existing code
class TieredDiscount(private val tiers: List<Tier>) : DiscountPolicy {
    override fun calculate(order: Order): Money =
        tiers.firstOrNull { it.matches(order) }?.discount ?: Money.ZERO
}

class DiscountCalculator(private val policy: DiscountPolicy) {
    fun applyDiscount(order: Order): Order =
        order.withDiscount(policy.calculate(order))
}
```

### Template Method Application

```kotlin
// Base class defines the algorithm skeleton
abstract class DataExporter {
    fun export(data: List<Record>): ByteArray {
        val filtered = filterRecords(data)
        val formatted = formatRecords(filtered)
        return encode(formatted)
    }

    protected open fun filterRecords(data: List<Record>): List<Record> = data
    protected abstract fun formatRecords(data: List<Record>): String
    protected abstract fun encode(content: String): ByteArray
}

// Extension: new format without modifying base class
class CsvExporter : DataExporter() {
    override fun formatRecords(data: List<Record>): String = ...
    override fun encode(content: String): ByteArray = content.toByteArray()
}
```

### Sealed Class for Bounded Extension

```kotlin
// When extension should be controlled, use sealed types
sealed interface PaymentMethod {
    data class CreditCard(val number: String, val expiry: String) : PaymentMethod
    data class BankTransfer(val accountNumber: String) : PaymentMethod
    data object Cash : PaymentMethod
}

// Exhaustive when — adding new subtype forces handling everywhere
fun process(method: PaymentMethod): PaymentResult = when (method) {
    is PaymentMethod.CreditCard -> processCreditCard(method)
    is PaymentMethod.BankTransfer -> processBankTransfer(method)
    is PaymentMethod.Cash -> processCash()
}
```

### OCP Rules

- Identify the axis of change first — then introduce abstraction at that point
- Do not over-abstract prematurely — apply OCP when a second variation actually appears
- Sealed types provide OCP with compile-time exhaustiveness for known, bounded variations
- Open interfaces provide OCP for unbounded, pluggable variations

---

## 3. LSP (Liskov Substitution Principle)

### LSP Core Concept

- Subtypes must be substitutable for their base types without altering program correctness
- If S is a subtype of T, then objects of type T can be replaced with objects of type S without breaking expectations
- A subtype must honor the **behavioral contract** of its supertype

### Subtype Rules

| Rule               | Description                                                            |
| ------------------ | ---------------------------------------------------------------------- |
| Precondition rule  | Subtypes must not strengthen preconditions                             |
| Postcondition rule | Subtypes must not weaken postconditions                                |
| Invariant rule     | Subtypes must preserve supertype invariants                            |
| History rule       | Subtypes must not introduce state changes the supertype does not allow |

### Violation Examples

```kotlin
// Classic violation: Square is NOT a behavioral subtype of Rectangle
open class Rectangle(open var width: Int, open var height: Int) {
    open fun area(): Int = width * height
}

class Square(side: Int) : Rectangle(side, side) {
    override var width: Int = side
        set(value) { field = value; height = value }  // Breaks independent width/height
    override var height: Int = side
        set(value) { field = value; width = value }
}

// Client code breaks: expects width and height to be independent
fun doubleWidth(rect: Rectangle) {
    rect.width *= 2
    check(rect.area() == rect.width * rect.height)  // Fails for Square
}
```

```kotlin
// Fix: model as separate types with shared interface
interface Shape {
    fun area(): Int
}

data class Rectangle(val width: Int, val height: Int) : Shape {
    override fun area(): Int = width * height
}

data class Square(val side: Int) : Shape {
    override fun area(): Int = side * side
}
```

### Design by Contract

```kotlin
interface Collection<E> {
    /**
     * Contract:
     * - Precondition: element is not null
     * - Postcondition: contains(element) returns true after add
     * - Postcondition: size increases by 1
     */
    fun add(element: E): Boolean
}

// Violation: ReadOnlyCollection breaks postcondition by throwing
class ReadOnlyCollection<E> : Collection<E> {
    override fun add(element: E): Boolean =
        throw UnsupportedOperationException()  // LSP violation
}
```

### LSP Rules

- Prefer composition over inheritance when behavioral substitution is not guaranteed
- Use interfaces to define contracts — verify all implementations satisfy the contract
- Throw-on-method implementations (e.g., `UnsupportedOperationException`) signal LSP violations
- If a subtype needs to disable supertype behavior, the inheritance hierarchy is wrong

---

## 4. ISP (Interface Segregation Principle)

### ISP Core Concept

- Clients should not be forced to depend on methods they do not use
- Split large interfaces into smaller, focused ones
- Each interface represents a **role** that a client cares about

### Violation and Fix

```kotlin
// Bad: fat interface forces all implementors to handle irrelevant methods
interface UserService {
    fun findById(id: Long): User
    fun findAll(pageable: Pageable): Page<User>
    fun create(request: CreateUserRequest): User
    fun update(id: Long, request: UpdateUserRequest): User
    fun delete(id: Long)
    fun exportToCsv(): ByteArray
    fun importFromCsv(data: ByteArray)
    fun sendVerificationEmail(userId: Long)
}

// Good: segregated by client needs
interface UserReader {
    fun findById(id: Long): User
    fun findAll(pageable: Pageable): Page<User>
}

interface UserWriter {
    fun create(request: CreateUserRequest): User
    fun update(id: Long, request: UpdateUserRequest): User
    fun delete(id: Long)
}

interface UserDataTransfer {
    fun exportToCsv(): ByteArray
    fun importFromCsv(data: ByteArray)
}

// A class can implement multiple role interfaces
class UserServiceImpl(
    private val repository: UserRepository,
    private val emailSender: EmailSender
) : UserReader, UserWriter {
    override fun findById(id: Long): User = ...
    override fun create(request: CreateUserRequest): User = ...
    // ...
}
```

### Client-Specific Interfaces

```kotlin
// Controller only needs read operations
class UserController(private val userReader: UserReader) {
    fun getUser(id: Long): UserResponse =
        userReader.findById(id).toResponse()
}

// Admin controller needs both read and write
class AdminUserController(
    private val userReader: UserReader,
    private val userWriter: UserWriter
) { ... }
```

### ISP Rules

- Design interfaces from the client's perspective, not the implementor's
- A class implementing many interfaces is acceptable — a client depending on a fat interface is not
- Prefer many small interfaces (3-5 methods) over few large ones
- When a single method interface suffices, consider `fun interface` for SAM conversion

---

## 5. DIP (Dependency Inversion Principle)

### DIP Core Concept

- High-level modules should not depend on low-level modules — both should depend on abstractions
- Abstractions should not depend on details — details should depend on abstractions
- The direction of source code dependency is **inverted** relative to the flow of control

### DIP Practical Application

```kotlin
// Bad: high-level policy depends on low-level detail
class OrderService {
    private val repository = MySqlOrderRepository()  // Concrete dependency
    private val notifier = SmtpEmailNotifier()       // Concrete dependency

    fun placeOrder(order: Order) {
        repository.save(order)
        notifier.notify(order)
    }
}

// Good: depend on abstractions defined by the high-level module
interface OrderRepository {
    fun save(order: Order): Order
    fun findById(id: Long): Order?
}

interface OrderNotifier {
    fun notify(order: Order)
}

class OrderService(
    private val repository: OrderRepository,   // Abstraction
    private val notifier: OrderNotifier         // Abstraction
) {
    fun placeOrder(order: Order) {
        repository.save(order)
        notifier.notify(order)
    }
}

// Low-level modules implement the abstractions
class JpaOrderRepository : OrderRepository { ... }
class EmailOrderNotifier : OrderNotifier { ... }
class SlackOrderNotifier : OrderNotifier { ... }
```

### Ports and Adapters Relationship

```text
Domain Layer (high-level)
├── OrderService (uses ports)
├── OrderRepository (port — outbound interface)
└── OrderNotifier (port — outbound interface)

Infrastructure Layer (low-level)
├── JpaOrderRepository (adapter — implements port)
├── EmailOrderNotifier (adapter — implements port)
└── SlackOrderNotifier (adapter — implements port)
```

- **Port**: interface defined in the domain layer (abstraction owned by high-level module)
- **Adapter**: implementation in the infrastructure layer (detail that depends on abstraction)
- DIP ensures the domain layer has zero dependency on infrastructure frameworks

### Dependency Injection

```kotlin
// Spring Boot: constructor injection wires adapters to ports
@Configuration
class InfraConfig {
    @Bean
    fun orderRepository(jpaRepository: JpaOrderJpaRepository): OrderRepository =
        JpaOrderRepository(jpaRepository)

    @Bean
    fun orderNotifier(emailSender: EmailSender): OrderNotifier =
        EmailOrderNotifier(emailSender)
}
```

### DIP Rules

- Abstractions (interfaces) belong in the **same module** as the high-level code that uses them
- Low-level modules depend on and implement the high-level module's abstractions
- Do not create interfaces for every class — apply DIP at architectural boundaries
- Stable abstractions change less frequently than volatile implementations

---

## 6. Additional Design Principles

### Tell, Don't Ask

```kotlin
// Bad: asking for state, then making decisions externally
if (account.balance >= amount) {
    account.balance -= amount
}

// Good: tell the object to perform the operation
account.withdraw(amount)  // Object decides internally
```

- Objects should encapsulate behavior, not just expose state
- Move decisions close to the data they depend on
- Getters are acceptable for display/reporting — but avoid using them for branching logic

### Law of Demeter (Principle of Least Knowledge)

```kotlin
// Bad: navigating through object chains
val city = order.customer.address.city

// Good: ask the immediate collaborator
val city = order.deliveryCity()
```

- A method should only call methods on:
  - Its own object (`this`)
  - Its parameters
  - Objects it creates
  - Its direct collaborators (fields)
- Chained calls like `a.b().c().d()` indicate missing encapsulation
- Exception: fluent APIs and builder patterns are acceptable chains

### Composition over Inheritance

| Aspect            | Inheritance                | Composition         |
| ----------------- | -------------------------- | ------------------- |
| Coupling          | Tight (white-box)          | Loose (black-box)   |
| Flexibility       | Static (compile-time)      | Dynamic (runtime)   |
| Reuse granularity | Entire class               | Individual behavior |
| Fragility         | Fragile base class problem | Isolated changes    |

```kotlin
// Prefer composition
class OrderValidator(
    private val rules: List<ValidationRule<Order>>
) {
    fun validate(order: Order): ValidationResult =
        rules.map { it.validate(order) }.merge()
}

// Over inheritance
abstract class BaseOrderValidator {
    abstract fun additionalRules(): List<ValidationRule<Order>>
    // Subclasses are coupled to base class internals
}
```

- Use inheritance only for genuine "is-a" relationships with shared behavior
- Prefer interfaces + delegation over abstract class hierarchies
- Kotlin's `by` keyword enables clean delegation without boilerplate

### Favor Immutability

```kotlin
// Immutable value object
data class Money(val amount: BigDecimal, val currency: Currency) {
    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Currency mismatch" }
        return Money(amount + other.amount, currency)
    }
}

// Immutable entity state transitions return new instances
data class Order private constructor(
    val id: OrderId,
    val status: OrderStatus,
    val items: List<OrderItem>
) {
    fun confirm(): Order = copy(status = OrderStatus.CONFIRMED)
    fun cancel(): Order = copy(status = OrderStatus.CANCELLED)
}
```

- Immutable objects are thread-safe, easier to reason about, and prevent accidental mutation
- Use `val` by default — `var` only when mutation is essential
- State transitions return new instances rather than modifying in place

---

## 7. Responsibility Assignment Patterns

### Information Expert

- Assign responsibility to the class that has the information needed to fulfill it

```kotlin
// Order has the items — it should calculate the total
class Order(val items: List<OrderItem>) {
    fun totalAmount(): Money = items.sumOf { it.subtotal() }
}

// OrderItem has quantity and price — it calculates subtotal
class OrderItem(val product: Product, val quantity: Int) {
    fun subtotal(): Money = product.price * quantity
}
```

### Creator

- Assign object creation to the class that has the initialization data

```kotlin
// Order creates OrderItem because it has the context
class Order {
    private val items = mutableListOf<OrderItem>()

    fun addItem(product: Product, quantity: Int): OrderItem {
        val item = OrderItem(product, quantity)
        items.add(item)
        return item
    }
}
```

### Low Coupling

- Minimize dependencies between classes to reduce the impact of change
- Prefer depending on stable abstractions over volatile implementations
- Reduce fan-out: each class should collaborate with a small number of others

### High Cohesion

- Keep related behavior together — a class's methods should operate on the same data
- If a subset of methods operates on a subset of fields, consider splitting the class
- High cohesion and SRP reinforce each other

---

## 8. Design Quality Metrics

### Coupling

| Type             | Description                                            | Strength          |
| ---------------- | ------------------------------------------------------ | ----------------- |
| Content coupling | One module modifies another's internals                | Strongest (worst) |
| Common coupling  | Modules share global mutable state                     | Strong            |
| Control coupling | One module controls another's flow via flags           | Moderate          |
| Stamp coupling   | Modules share a data structure but use different parts | Moderate          |
| Data coupling    | Modules share only necessary data via parameters       | Weak (best)       |

### Cohesion

| Type            | Description                                | Strength         |
| --------------- | ------------------------------------------ | ---------------- |
| Functional      | All elements contribute to a single task   | Strongest (best) |
| Sequential      | Output of one element is input to the next | Strong           |
| Communicational | Elements operate on the same data          | Moderate         |
| Temporal        | Elements are related by timing             | Weak             |
| Coincidental    | No meaningful relationship                 | Weakest (worst)  |

### Fan-In and Fan-Out

| Metric  | Definition                                  | Guideline                                            |
| ------- | ------------------------------------------- | ---------------------------------------------------- |
| Fan-in  | Number of classes that depend on this class | High fan-in is acceptable for stable utilities       |
| Fan-out | Number of classes this class depends on     | High fan-out (>7) indicates excessive responsibility |

- High fan-in + low fan-out = stable, reusable module
- Low fan-in + high fan-out = orchestrator or mediator (acceptable for facades)
- High fan-in + high fan-out = risky bottleneck — refactor to reduce responsibilities

---

## 9. Anti-Patterns

### God Class

- One class that knows too much and does too much
- Symptoms: 1000+ lines, 20+ methods, 10+ fields, many unrelated responsibilities
- Fix: decompose into focused classes using SRP

### Anemic Domain Model

- Domain objects contain only data (getters/setters) with no behavior
- Business logic lives entirely in service classes
- Fix: move behavior into domain objects following Information Expert

```kotlin
// Anemic: all logic in service
class OrderService {
    fun cancel(order: Order) {
        if (order.status == OrderStatus.SHIPPED) throw IllegalStateException()
        order.status = OrderStatus.CANCELLED
    }
}

// Rich: behavior in domain object
class Order(var status: OrderStatus) {
    fun cancel() {
        require(status != OrderStatus.SHIPPED) { "Cannot cancel shipped order" }
        status = OrderStatus.CANCELLED
    }
}
```

### Inappropriate Intimacy

- Two classes excessively access each other's internal details
- Symptoms: frequent access to private/internal members, bidirectional dependencies
- Fix: extract shared logic into a new class, or merge if truly one responsibility

### Refused Bequest

- Subclass inherits methods or properties it does not need or cannot support
- Symptoms: overriding methods to throw `UnsupportedOperationException`, empty implementations
- Fix: replace inheritance with composition, or redesign the hierarchy

### Feature Envy

- A method that uses more data from another class than its own
- Symptoms: long chains of getter calls on a single external object
- Fix: move the method to the class whose data it primarily uses

### Primitive Obsession

- Using primitive types (String, Int, Long) for domain concepts
- Symptoms: validation logic scattered across multiple locations

```kotlin
// Bad: email as String everywhere
fun sendEmail(to: String) { ... }  // No validation guarantee

// Good: value class with validation
@JvmInline
value class Email(val value: String) {
    init { require(value.matches(EMAIL_REGEX)) { "Invalid email: $value" } }
}

fun sendEmail(to: Email) { ... }  // Always valid
```

---

## 10. Related Rules

- **Code quality fundamentals**: `~/.claude/rules/code-quality.md`
- **Java conventions and patterns**: `~/.claude/rules/java-convention.md`
- **Kotlin conventions and patterns**: `~/.claude/rules/kotlin-convention.md`
- **Spring Framework patterns**: `~/.claude/rules/spring-framework.md`
- **Error handling design**: `~/.claude/rules/error-handling.md`
- **BDD test rules (testing behavior, not implementation)**: `~/.claude/rules/testing-unit.md`

---

## 11. Further Reading

- Robert C. Martin — "Clean Architecture" (SOLID principles in context)
- Craig Larman — "Applying UML and Patterns" (GRASP patterns)
- Martin Fowler — "Refactoring" (identifying and fixing design smells)
- Eric Evans — "Domain-Driven Design" (domain modeling and responsibility)
- Joshua Bloch — "Effective Java" (immutability, composition, API design)
- Bertrand Meyer — "Object-Oriented Software Construction" (Design by Contract)
