---
name: exposed
description: >-
  Jetbrains Exposed ORM rules including DSL vs DAO, table definition,
  query patterns, transaction management, Spring Boot integration,
  and schema migration. Use when writing Exposed ORM code.
---

# Exposed ORM Rules

## 1. DSL vs DAO

### API Comparison

| Aspect       | DSL (Typesafe SQL)                   | DAO (Active Record)                |
| ------------ | ------------------------------------ | ---------------------------------- |
| Style        | Functional, query-builder            | Object-oriented, entity-based      |
| Best for     | Complex queries, reporting           | CRUD operations, domain modeling   |
| Type safety  | Column-level                         | Entity-level                       |
| Flexibility  | Full SQL expressiveness              | Simpler but less flexible          |
| Caching      | No                                   | Entity-level caching available     |

### When to Use Which

- Use **DSL** for complex joins, aggregations, subqueries, and read-heavy operations
- Use **DAO** for simple CRUD with entity lifecycle and relationship management
- Mixing both in the same project is acceptable — use DSL for queries, DAO for mutations

---

## 2. Table Definition

### DSL Table Definition

```kotlin
object Users : LongIdTable("users") {
    val name = varchar("name", 100)
    val email = varchar("email", 255).uniqueIndex()
    val status = enumerationByName<UserStatus>("status", 20)
    val createdAt = timestamp("created_at").defaultExpression(CurrentTimestamp)
    val updatedAt = timestamp("updated_at").defaultExpression(CurrentTimestamp)
}

object Orders : LongIdTable("orders") {
    val userId = reference("user_id", Users)
    val amount = decimal("amount", 10, 2)
    val status = enumerationByName<OrderStatus>("status", 20)
    val createdAt = timestamp("created_at").defaultExpression(CurrentTimestamp)
}
```

### DAO Entity Definition

```kotlin
class UserEntity(id: EntityID<Long>) : LongEntity(id) {
    companion object : LongEntityClass<UserEntity>(Users)

    var name by Users.name
    var email by Users.email
    var status by Users.status
    var createdAt by Users.createdAt
    var updatedAt by Users.updatedAt

    val orders by OrderEntity referrersOn Orders.userId
}

class OrderEntity(id: EntityID<Long>) : LongEntity(id) {
    companion object : LongEntityClass<OrderEntity>(Orders)

    var user by UserEntity referencedOn Orders.userId
    var amount by Orders.amount
    var status by Orders.status
    var createdAt by Orders.createdAt
}
```

### Table Definition Rules

- Use `LongIdTable` or `UUIDTable` for auto-generated ID tables
- Use `Table` for join tables or tables with composite keys
- Use `enumerationByName` over `enumeration` (stores string, not ordinal)
- Use `reference()` for foreign keys — it creates the column and constraint
- Define `uniqueIndex()` on business-unique columns (email, etc.)

---

## 3. DSL Query Patterns

### Basic CRUD

```kotlin
// Insert
val userId = Users.insertAndGetId {
    it[name] = "John"
    it[email] = "john@example.com"
    it[status] = UserStatus.ACTIVE
}

// Select with conditions
val activeUsers = Users
    .selectAll()
    .where { (Users.status eq UserStatus.ACTIVE) and (Users.createdAt greater cutoff) }
    .map { row -> UserResponse(row[Users.id].value, row[Users.name], row[Users.email]) }

// Update
Users.update({ Users.id eq userId }) {
    it[status] = UserStatus.INACTIVE
    it[updatedAt] = Instant.now()
}

// Delete
Users.deleteWhere { Users.id eq userId }
```

### Join Queries

```kotlin
// Inner join
val result = Users
    .innerJoin(Orders)
    .select(Users.name, Orders.amount, Orders.status)
    .where { Orders.status eq OrderStatus.COMPLETED }
    .map { row -> OrderSummary(row[Users.name], row[Orders.amount]) }

// Left join with alias
val orderCount = Orders.id.count()
val summary = Users
    .leftJoin(Orders)
    .select(Users.name, orderCount)
    .groupBy(Users.id)
    .map { row -> UserSummary(row[Users.name], row[orderCount]) }
```

### Batch Operations

```kotlin
// Batch insert
Users.batchInsert(userList) { user ->
    this[Users.name] = user.name
    this[Users.email] = user.email
    this[Users.status] = UserStatus.ACTIVE
}

// Upsert (insert or update)
Users.upsert(Users.email) {
    it[name] = "John"
    it[email] = "john@example.com"
    it[status] = UserStatus.ACTIVE
}
```

---

## 4. Transaction Management

### Basic Transaction

```kotlin
// All database operations must run inside a transaction
transaction {
    val user = Users.insertAndGetId {
        it[name] = "John"
        it[email] = "john@example.com"
    }
    Orders.insert {
        it[userId] = user
        it[amount] = BigDecimal("100.00")
        it[status] = OrderStatus.PENDING
    }
}
```

### Transaction Configuration

```kotlin
// Read-only transaction (set at connection level)
transaction {
    connection.readOnly = true
    Users.selectAll().where { Users.status eq UserStatus.ACTIVE }.toList()
}

// Custom isolation level
transaction(transactionIsolation = Connection.TRANSACTION_REPEATABLE_READ) {
    // Critical business logic
}

// Nested transaction (savepoint)
transaction {
    // outer transaction
    val result = transaction {
        // inner transaction (savepoint)
        Users.insertAndGetId { it[name] = "John" }
    }
}
```

### Transaction Rules

- Every database operation must be wrapped in a `transaction` block
- Use `readOnly = true` for read-only operations
- Avoid long-running transactions — keep them short
- Never call external APIs inside a transaction
- Use nested transactions (savepoints) sparingly

---

## 5. Spring Boot Integration

### Dependency Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.jetbrains.exposed:exposed-spring-boot-starter:${exposedVersion}")
    implementation("org.jetbrains.exposed:exposed-core:${exposedVersion}")
    implementation("org.jetbrains.exposed:exposed-dao:${exposedVersion}")
    implementation("org.jetbrains.exposed:exposed-kotlin-datetime:${exposedVersion}")
    implementation("org.jetbrains.exposed:exposed-json:${exposedVersion}")
}
```

### Spring Configuration

```yaml
# application.yml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

exposed:
  generate-ddl: false
  show-sql: ${EXPOSED_SHOW_SQL:false}
```

### Spring Transaction Integration

```kotlin
@Service
class UserService(
    private val database: Database
) {
    @Transactional(readOnly = true)
    fun findActiveUsers(): List<UserResponse> = transaction {
        Users.selectAll()
            .where { Users.status eq UserStatus.ACTIVE }
            .map { it.toUserResponse() }
    }

    @Transactional
    fun createUser(request: CreateUserRequest): Long = transaction {
        Users.insertAndGetId {
            it[name] = request.name
            it[email] = request.email
            it[status] = UserStatus.ACTIVE
        }.value
    }
}
```

### Spring Integration Rules

- Use `exposed-spring-boot-starter` for auto-configuration
- Spring `@Transactional` and Exposed `transaction` can coexist — Spring manages the outer transaction
- Set `exposed.generate-ddl=false` in production — use migration tools instead
- Use `exposed-kotlin-datetime` for `kotlinx.datetime` type support

---

## 6. Schema Migration

### Migration Tool Options

| Tool      | Integration           | Use Case                           |
| --------- | --------------------- | ---------------------------------- |
| Flyway    | Spring Boot native    | SQL-based migrations (recommended) |
| Liquibase | Spring Boot native    | XML/YAML/SQL migrations            |
| Exposed   | `SchemaUtils.create`  | Development/testing only           |

### Migration Rules

- Never use `SchemaUtils.create` or `SchemaUtils.createMissingTablesAndColumns` in production
- Use Flyway or Liquibase for versioned, repeatable migrations
- Write migration SQL manually — do not rely on auto-generation for production schemas
- Test migrations against production-like data before applying

---

## 7. Anti-Patterns

- Using `SchemaUtils` for production schema management
- Running queries outside a transaction block
- Using `enumeration` (ordinal-based) instead of `enumerationByName` (string-based)
- Fetching all columns when only a few are needed in DSL queries
- Not using `batchInsert` for bulk operations (N individual inserts)
- Mixing DSL and DAO for the same table without clear separation
- Not setting `readOnly = true` for read operations
- Using Exposed entity objects outside of transaction scope (lazy loading fails)
