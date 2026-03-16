---
name: spring-jpa
description: >-
  Spring Boot JPA patterns including N+1 prevention, @Transactional,
  JPA entity conventions, and Spring Data repository patterns.
  Use when writing JPA entities or Spring Data repositories.
---

# Spring Boot Database Rules

## 1. N+1 Problem Prevention

```kotlin
// Bad: N+1 queries
val users = userRepository.findAll()
users.forEach { user ->
    val orders = orderRepository.findByUserId(user.id) // N additional queries
}

// Good: Fetch join
@Query("SELECT u FROM User u JOIN FETCH u.orders")
fun findAllWithOrders(): List<User>

// Good: EntityGraph
@EntityGraph(attributePaths = ["orders"])
fun findAll(): List<User>
```

---

## 2. Spring Transaction Management

### Transactional Service Example

```kotlin
@Transactional
fun createOrder(request: CreateOrderRequest): Order {
    val user = userRepository.findByIdOrThrow(request.userId)
    val order = Order.create(user, request.items)
    return orderRepository.save(order)
}

// External API calls should NOT be inside a transaction
fun processOrder(orderId: Long) {
    val order = orderService.findById(orderId) // read
    val result = paymentClient.charge(order)    // external call — outside tx
    orderService.updatePaymentResult(order, result) // separate tx
}
```

### Spring Transaction Rules

| Rule                                             | Reason                                 |
| ------------------------------------------------ | -------------------------------------- |
| Use `@Transactional(readOnly = true)` for reads  | Enables query optimizations            |
| Default propagation is `REQUIRED`                | Reuses existing transaction            |
| Use `REQUIRES_NEW` sparingly                     | Creates independent tx — deadlock risk |
| Never call external APIs inside `@Transactional` | Prevents long-held locks               |

---

## 3. JPA Entity Conventions

```kotlin
@Entity
@Table(name = "users")
class User(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(nullable = false, length = 100)
    var name: String,

    @Column(nullable = false, unique = true)
    var email: String,

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    var status: UserStatus = UserStatus.ACTIVE,

    @Column(nullable = false, updatable = false)
    val createdAt: Instant = Instant.now(),

    @Column(nullable = false)
    var updatedAt: Instant = Instant.now()
) {
    fun deactivate() {
        status = UserStatus.INACTIVE
        updatedAt = Instant.now()
    }
}
```

### Entity Design Rules

- Use `val` for immutable fields (id, createdAt), `var` for mutable fields
- Always use `EnumType.STRING` for enums (not `ORDINAL`)
- Include `createdAt` and `updatedAt` audit columns
- Put business logic in entity methods, not in service layer
- Avoid bidirectional relationships unless necessary — prefer unidirectional

---

## 4. Spring Data Repository Patterns

```kotlin
interface UserRepository : JpaRepository<User, Long> {
    fun findByEmail(email: String): User?
    fun findByStatusAndCreatedAtAfter(status: UserStatus, after: Instant): List<User>
    fun existsByEmail(email: String): Boolean

    @Query("SELECT u FROM User u WHERE u.name LIKE CONCAT(:keyword, '%')")
    fun searchByNamePrefix(@Param("keyword") keyword: String, pageable: Pageable): Page<User>
}

// Extension for throwing on not found
fun UserRepository.findByIdOrThrow(id: Long): User =
    findByIdOrNull(id) ?: throw EntityNotFoundException("User", id)
```

### Repository Rules

- Use Spring Data derived query methods for simple queries
- Use `@Query` with JPQL for complex queries
- Use native queries (`nativeQuery = true`) only when JPQL is insufficient
- Always use `Pageable` for list queries that may return large result sets
