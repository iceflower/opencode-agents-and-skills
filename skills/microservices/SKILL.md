# Microservices Architecture Pattern Rules

## 1. Service Decomposition Principles

### Decomposition Criteria

| Criterion           | Description                                           | Example                              |
| ------------------- | ----------------------------------------------------- | ------------------------------------ |
| Business Capability | Align services with organizational business functions | Order, Payment, Shipping, Inventory  |
| Bounded Context     | DDD aggregate boundaries define service scope         | User ≠ Customer (different contexts) |
| Team Autonomy       | One team owns one or more services end-to-end         | Team can deploy independently        |
| Data Ownership      | Each service owns its data exclusively                | Order DB, Payment DB separated       |

### Decomposition Rules

- A service should be deployable and scalable independently
- A service should represent a single business capability or bounded context
- If two services always change together, they should be one service
- If a service requires deep knowledge of another service's internals, the boundary is wrong
- Start with fewer, coarser services — split only when complexity demands it

### Right-Sizing a Service

| Signal                                   | Action                   |
| ---------------------------------------- | ------------------------ |
| Service has too many responsibilities    | Split by business domain |
| Two services always deploy together      | Merge into one           |
| Team cannot understand the full codebase | Consider splitting       |
| Service has only CRUD operations         | May be too granular      |
| Cross-service transactions are frequent  | Reconsider boundaries    |

---

## 2. Communication Patterns

### Synchronous Communication

| Protocol | Strengths                          | Weaknesses                               | Use Case                    |
| -------- | ---------------------------------- | ---------------------------------------- | --------------------------- |
| REST     | Simple, ubiquitous, human-readable | Higher latency, no streaming             | CRUD APIs, public-facing    |
| gRPC     | Fast, typed, streaming support     | Requires proto definitions, less tooling | Internal service-to-service |

### Asynchronous Communication

| Pattern           | Description                     | Use Case                           |
| ----------------- | ------------------------------- | ---------------------------------- |
| Message Queue     | Point-to-point delivery         | Task delegation, work distribution |
| Publish/Subscribe | Broadcast to multiple consumers | Event notification, data sync      |
| Event Streaming   | Ordered, replayable event log   | Event sourcing, audit trail        |

### Communication Selection Criteria

| Scenario                           | Recommended              |
| ---------------------------------- | ------------------------ |
| Need immediate response            | Synchronous (REST/gRPC)  |
| Fire-and-forget operation          | Async messaging          |
| Multiple consumers need same event | Pub/Sub                  |
| Ordering and replay required       | Event streaming          |
| Long-running operation             | Async + callback/polling |
| Cross-service data consistency     | Saga via messaging       |

### Communication Patterns Rules

- Prefer asynchronous communication between services — it reduces temporal coupling
- Use synchronous calls only when the caller needs an immediate response
- Never chain more than two synchronous calls — use async or aggregation instead
- Always set timeouts on synchronous calls
- Design messages to be self-contained — consumers should not need to call back to the producer

### Spring Boot Example: Event Publishing

```kotlin
// Domain event
data class OrderCreatedEvent(
    val orderId: String,
    val userId: String,
    val totalAmount: BigDecimal,
    val items: List<OrderItem>,
    val occurredAt: Instant = Instant.now()
)

// Publishing service
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val eventPublisher: OrderEventPublisher
) {
    @Transactional
    fun createOrder(request: CreateOrderRequest): Order {
        val order = orderRepository.save(Order.from(request))
        eventPublisher.publish(order.toCreatedEvent())
        return order
    }
}
```

---

## 3. API Gateway Pattern

### Gateway Responsibilities

| Responsibility       | Description                                    |
| -------------------- | ---------------------------------------------- |
| Routing              | Route requests to appropriate backend services |
| Authentication       | Validate tokens, enforce identity              |
| Rate Limiting        | Protect backends from traffic spikes           |
| Response Aggregation | Combine responses from multiple services       |
| Protocol Translation | REST to gRPC, WebSocket to HTTP, etc.          |
| Load Balancing       | Distribute traffic across service instances    |
| Caching              | Cache frequently requested responses           |

### Gateway Rules

- The gateway should NOT contain business logic — only cross-cutting concerns
- Use a single gateway for external clients; consider per-client gateways (BFF) for different client types
- Rate limiting should be applied at the gateway level, not in each service
- Authentication should happen at the gateway; authorization should happen in individual services
- Circuit breakers at the gateway protect against cascading failures from unhealthy backends

### Backend for Frontend (BFF) Pattern

```text
Mobile App  → Mobile BFF  → Internal Services
Web App     → Web BFF     → Internal Services
Partner API → Partner BFF → Internal Services
```

- Each BFF is tailored to its client's specific needs
- BFF aggregates and transforms backend responses for its client
- Avoids one-size-fits-all API that serves no client well

### Anti-Patterns

- Gateway becoming a monolith with business logic
- Single point of failure without redundancy
- Gateway performing data transformation that belongs in services
- Skipping the gateway for "internal" calls from external clients

---

## 4. Service Discovery

### Discovery Models

| Model       | How It Works                                   | Example                     |
| ----------- | ---------------------------------------------- | --------------------------- |
| Client-Side | Client queries registry, picks instance        | Eureka + Ribbon             |
| Server-Side | Load balancer queries registry, routes request | Kubernetes Service, AWS ALB |
| DNS-Based   | Service registers DNS record, client resolves  | Consul DNS, CoreDNS         |

### Comparison

| Aspect               | Client-Side      | Server-Side       | DNS-Based  |
| -------------------- | ---------------- | ----------------- | ---------- |
| Client complexity    | High (LB logic)  | Low (transparent) | Low        |
| Infrastructure needs | Service registry | LB + registry     | DNS server |
| Health checking      | Client-driven    | LB-driven         | TTL-based  |
| Kubernetes native    | No               | Yes               | Yes        |

### Service Discovery Rules

- In Kubernetes environments, use native Service resources — no external registry needed
- For non-Kubernetes environments, use a dedicated service registry (Consul, Eureka)
- Always implement health checks — unhealthy instances must be removed from discovery
- Use DNS-based discovery for simplicity when advanced load balancing is not required
- Set appropriate TTL for DNS records to balance freshness and DNS load

---

## 5. Distributed Transactions (Saga Pattern)

### Why Distributed Transactions

- Each service owns its own database — no shared transactions across services
- Two-phase commit (2PC) has high latency, tight coupling, and poor availability — avoid it
- Saga pattern achieves eventual consistency through a sequence of local transactions

### Choreography vs Orchestration

| Aspect                  | Choreography                      | Orchestration                     |
| ----------------------- | --------------------------------- | --------------------------------- |
| Coordination            | Each service listens and reacts   | Central orchestrator directs flow |
| Coupling                | Low (event-driven)                | Medium (orchestrator knows steps) |
| Complexity              | Grows with number of participants | Centralized, easier to trace      |
| Debugging               | Hard to follow event chain        | Single point to observe flow      |
| Single point of failure | None                              | Orchestrator                      |
| Best for                | Simple flows (2-4 services)       | Complex flows (5+ services)       |

### Choreography Example

```text
Order Service → [OrderCreated] → Payment Service
Payment Service → [PaymentCompleted] → Inventory Service
Inventory Service → [InventoryReserved] → Shipping Service

On failure:
Inventory Service → [ReservationFailed] → Payment Service (refund)
Payment Service → [PaymentRefunded] → Order Service (cancel)
```

### Orchestration Example

```kotlin
@Service
class OrderSagaOrchestrator(
    private val paymentClient: PaymentClient,
    private val inventoryClient: InventoryClient,
    private val shippingClient: ShippingClient,
    private val orderRepository: OrderRepository
) {
    suspend fun execute(order: Order) {
        try {
            val payment = paymentClient.charge(order.toPaymentRequest())
            val reservation = inventoryClient.reserve(order.toReservationRequest())
            shippingClient.schedule(order.toShippingRequest())
            orderRepository.updateStatus(order.id, OrderStatus.CONFIRMED)
        } catch (e: PaymentException) {
            orderRepository.updateStatus(order.id, OrderStatus.CANCELLED)
        } catch (e: InventoryException) {
            paymentClient.refund(order.paymentId)
            orderRepository.updateStatus(order.id, OrderStatus.CANCELLED)
        } catch (e: ShippingException) {
            inventoryClient.release(order.reservationId)
            paymentClient.refund(order.paymentId)
            orderRepository.updateStatus(order.id, OrderStatus.CANCELLED)
        }
    }
}
```

### Compensating Transactions

| Forward Action    | Compensating Action |
| ----------------- | ------------------- |
| Charge payment    | Refund payment      |
| Reserve inventory | Release inventory   |
| Schedule shipment | Cancel shipment     |
| Create order      | Cancel order        |

### Saga Rules

- Every forward action must have a corresponding compensating action
- Compensating actions must be idempotent — they may be invoked multiple times
- Design for eventual consistency — intermediate states are visible to users
- Use unique saga IDs for tracing the entire saga lifecycle
- Persist saga state to handle orchestrator failures and restarts
- Prefer choreography for simple flows; switch to orchestration when flows become complex

---

## 6. CQRS and Event Sourcing

### CQRS (Command Query Responsibility Segregation)

```text
Client → Command API → Write Model → Event Store
Client → Query API  → Read Model  ← Projection (from events)
```

### When to Use CQRS

| Use CQRS                                      | Avoid CQRS                         |
| --------------------------------------------- | ---------------------------------- |
| Read and write workloads differ significantly | Simple CRUD with uniform access    |
| Complex domain with rich business rules       | Small team or early-stage project  |
| Need different read models for different UIs  | Data consistency must be immediate |
| High read-to-write ratio                      | Domain model is straightforward    |

### Event Sourcing

```kotlin
// Event as the source of truth
sealed class OrderEvent {
    data class Created(val orderId: String, val items: List<Item>) : OrderEvent()
    data class ItemAdded(val orderId: String, val item: Item) : OrderEvent()
    data class Confirmed(val orderId: String, val confirmedAt: Instant) : OrderEvent()
    data class Cancelled(val orderId: String, val reason: String) : OrderEvent()
}

// Rebuild state from events
fun Order.Companion.fromEvents(events: List<OrderEvent>): Order {
    return events.fold(Order.empty()) { state, event ->
        when (event) {
            is OrderEvent.Created -> state.copy(id = event.orderId, items = event.items)
            is OrderEvent.ItemAdded -> state.copy(items = state.items + event.item)
            is OrderEvent.Confirmed -> state.copy(status = OrderStatus.CONFIRMED)
            is OrderEvent.Cancelled -> state.copy(status = OrderStatus.CANCELLED)
        }
    }
}
```

### Event Sourcing Trade-Offs

| Advantage                            | Disadvantage                        |
| ------------------------------------ | ----------------------------------- |
| Complete audit trail                 | Increased storage requirements      |
| Temporal queries (state at any time) | Complex event schema evolution      |
| Natural fit for event-driven arch    | Eventually consistent read models   |
| Debugging via event replay           | Learning curve for development team |

### CQRS and Event Sourcing Rules

- CQRS and Event Sourcing are independent patterns — use one without the other
- Do not apply CQRS to the entire system — use it where read/write asymmetry exists
- Event Sourcing requires an event schema evolution strategy from day one
- Use snapshots to avoid replaying long event histories on every read
- Read model projections should be rebuildable from the event log at any time

---

## 7. Data Management

### Database per Service

```text
Order Service  → Order DB  (PostgreSQL)
Payment Service → Payment DB (PostgreSQL)
Search Service  → Search Index (Elasticsearch)
Cache Service   → Cache Store (Redis)
```

### Data Ownership Rules

- Each service owns its database exclusively — no other service accesses it directly
- Services expose data through APIs, not through shared database access
- Use events to propagate data changes to other services that need them
- Each service can choose the database technology best suited to its needs (polyglot persistence)

### Data Synchronization Patterns

| Pattern             | Description                                | Consistency    | Complexity |
| ------------------- | ------------------------------------------ | -------------- | ---------- |
| Event-Driven Sync   | Publish events on change, consumers update | Eventual       | Medium     |
| Change Data Capture | Capture DB changes from transaction log    | Near real-time | Medium     |
| API Polling         | Periodically fetch from source service     | Delayed        | Low        |
| Dual Write          | Write to both DB and event store           | Risky          | Low        |

### Transactional Outbox Pattern

```kotlin
// Write to DB and outbox table in same transaction
@Transactional
fun createOrder(request: CreateOrderRequest): Order {
    val order = orderRepository.save(Order.from(request))

    // Outbox entry — same transaction as business write
    outboxRepository.save(OutboxEntry(
        aggregateType = "Order",
        aggregateId = order.id,
        eventType = "OrderCreated",
        payload = objectMapper.writeValueAsString(order.toEvent())
    ))

    return order
}

// Separate process polls outbox and publishes to message broker
// After successful publish, mark outbox entry as published
```

### Data Management Rules

- Never use dual write (writing to DB and message broker separately) — it causes inconsistency on partial failure
- Use Transactional Outbox or Change Data Capture for reliable event publishing
- Accept eventual consistency — design UIs and APIs to handle intermediate states gracefully
- Shared Database is an anti-pattern — it creates tight coupling and prevents independent deployment

---

## 8. Fault Isolation

### Resilience Patterns

| Pattern         | Purpose                               | Implementation                       |
| --------------- | ------------------------------------- | ------------------------------------ |
| Circuit Breaker | Stop calling a failing service        | Resilience4j, Spring Circuit Breaker |
| Bulkhead        | Limit concurrent calls per service    | Thread pool isolation, semaphore     |
| Retry           | Retry transient failures with backoff | Spring Retry, Resilience4j           |
| Timeout         | Fail fast when service is slow        | HTTP client timeout, `withTimeout`   |
| Fallback        | Provide degraded response on failure  | Default value, cached response       |
| Rate Limiter    | Limit outbound request rate           | Token bucket, sliding window         |

### Circuit Breaker States

```text
CLOSED → (failure rate exceeds threshold) → OPEN
OPEN → (wait duration expires) → HALF_OPEN
HALF_OPEN → (test calls succeed) → CLOSED
HALF_OPEN → (test calls fail) → OPEN
```

### Resilience Configuration Example

```kotlin
@Service
class PaymentService(
    private val paymentClient: PaymentClient
) {
    @CircuitBreaker(name = "paymentApi", fallbackMethod = "paymentFallback")
    @Retry(name = "paymentApi")
    @Bulkhead(name = "paymentApi")
    fun processPayment(request: PaymentRequest): PaymentResponse {
        return paymentClient.charge(request)
    }

    fun paymentFallback(request: PaymentRequest, e: Exception): PaymentResponse {
        log.warn("Payment service unavailable, queuing for retry", e)
        pendingPaymentQueue.enqueue(request)
        return PaymentResponse.pending(request.orderId)
    }
}
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      paymentApi:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
  retry:
    instances:
      paymentApi:
        max-attempts: 3
        wait-duration: 1s
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.net.ConnectException
          - java.net.SocketTimeoutException
  bulkhead:
    instances:
      paymentApi:
        max-concurrent-calls: 10
        max-wait-duration: 500ms
```

### Resilience Rules

- Apply circuit breakers to all external service calls — no exceptions
- Set timeouts shorter than the caller's timeout to avoid cascading delays
- Use exponential backoff with jitter for retries — avoid thundering herd
- Retry only idempotent operations — or use idempotency keys for non-idempotent ones
- Bulkhead isolates failures — a slow service should not consume all threads
- Fallbacks should provide degraded but useful responses, not error pages
- Monitor circuit breaker state transitions — frequent OPEN states indicate systemic issues

---

## 9. Monolith to Microservices Migration

### Strangler Fig Pattern

```text
Phase 1: Route all traffic through new gateway
┌──────────┐     ┌──────────────┐
│  Gateway  │────→│  Monolith    │
└──────────┘     └──────────────┘

Phase 2: Extract one service, route specific paths to it
┌──────────┐     ┌──────────────┐
│  Gateway  │──┬─→│  Monolith    │  (remaining features)
└──────────┘  │  └──────────────┘
              │  ┌──────────────┐
              └─→│  Order Svc   │  (extracted feature)
                 └──────────────┘

Phase 3: Repeat until monolith is empty
```

### Branch by Abstraction

```kotlin
// Step 1: Introduce abstraction layer in monolith
interface NotificationSender {
    fun send(userId: String, message: String)
}

// Step 2: Old implementation (still in monolith)
class MonolithNotificationSender : NotificationSender {
    override fun send(userId: String, message: String) {
        // Direct DB call within monolith
    }
}

// Step 3: New implementation (calls extracted service)
class MicroserviceNotificationSender(
    private val notificationClient: NotificationClient
) : NotificationSender {
    override fun send(userId: String, message: String) {
        notificationClient.send(SendRequest(userId, message))
    }
}

// Step 4: Feature toggle to switch implementations
@Bean
fun notificationSender(
    @Value("\${feature.notification-service.enabled}") enabled: Boolean,
    notificationClient: NotificationClient
): NotificationSender =
    if (enabled) MicroserviceNotificationSender(notificationClient)
    else MonolithNotificationSender()
```

### Migration Strategy Rules

| Step | Action                                          | Risk   |
| ---- | ----------------------------------------------- | ------ |
| 1    | Identify bounded contexts in monolith           | Low    |
| 2    | Add API gateway in front of monolith            | Low    |
| 3    | Extract the least-coupled, highest-value domain | Medium |
| 4    | Migrate data to new service's database          | High   |
| 5    | Route traffic to new service via gateway        | Medium |
| 6    | Decommission extracted code from monolith       | Low    |
| 7    | Repeat for next domain                          | Varies |

### Monolith to Microservices Migration Rules

- Extract one service at a time — never do a big-bang rewrite
- Start with the domain that has the most to gain from independent scaling or deployment
- Maintain backward compatibility during migration — old and new must coexist
- Use feature toggles to switch between monolith and microservice implementations gradually
- Data migration is the hardest part — plan for dual-write or CDC during transition
- Keep the monolith running until the extracted service is proven in production

---

## 10. Anti-Patterns

### Distributed Monolith

- Services are deployed independently but must be deployed together due to tight coupling
- **Symptoms**: changing one service requires changes in multiple other services; shared libraries with business logic; synchronous call chains
- **Fix**: enforce service boundaries through API contracts; eliminate shared domain libraries; use async communication

### Excessive Service Decomposition

- Too many fine-grained services create operational overhead without business benefit
- **Symptoms**: services with only 1-2 endpoints; most calls are service-to-service, not from clients; team manages more services than it can handle
- **Fix**: merge related services; apply the "two-pizza team" rule; split only when complexity demands it

### Synchronous Call Chains

- Service A calls B, B calls C, C calls D — latency compounds, availability drops exponentially
- **Symptoms**: response time is sum of all services; one slow service degrades the entire chain; cascading failures
- **Fix**: use async messaging; aggregate data at the gateway; cache intermediate results; use CQRS for read-heavy paths

### Shared Database

- Multiple services read from and write to the same database
- **Symptoms**: schema changes require coordinating multiple teams; database becomes the bottleneck; services cannot be deployed independently
- **Fix**: migrate to database-per-service; use events for data synchronization; accept eventual consistency

### Missing Idempotency

- Consumers process the same message multiple times with different results
- **Symptoms**: duplicate orders, double charges, inconsistent state after retries
- **Fix**: use idempotency keys; store processed message IDs; design consumers to handle duplicates

### God Service

- One service accumulates too many responsibilities and becomes a new monolith
- **Symptoms**: service has dozens of endpoints spanning multiple domains; multiple teams contribute to the same service; deployment is risky due to scope
- **Fix**: decompose by bounded context; enforce single responsibility at the service level

---

## 11. Related Rule References

| Topic                 | Rule File                  | Relevance                                          |
| --------------------- | -------------------------- | -------------------------------------------------- |
| Messaging patterns    | `messaging.md`             | Broker selection, producer/consumer patterns       |
| API client resilience | `http-client.md`            | Timeout, retry, circuit breaker configuration      |
| Spring API client     | `spring-http-client.md`     | RestClient, Spring Retry, Resilience4j setup       |
| Error handling        | `error-handling.md`        | Exception hierarchy, error response format         |
| Spring error handling | `spring-error-handling.md` | @ControllerAdvice, ErrorCode enum                  |
| Monitoring            | `monitoring.md`            | Metrics, tracing, alerting for distributed systems |
| Spring monitoring     | `spring-monitoring.md`     | Actuator, Micrometer, distributed tracing          |
| Caching               | `caching.md`               | Cache strategy, TTL, invalidation patterns         |
| Database              | `database.md`              | Migration, transaction management, query patterns  |
| API design            | `api-design.md`            | REST conventions, versioning, pagination           |
| Security              | `security.md`              | Authentication, authorization, rate limiting       |
| Logging               | `logging.md`               | Structured logging, traceId correlation            |

---

## Further Reading

- Sam Newman, *Building Microservices* (2nd Edition, O'Reilly)
- Chris Richardson, *Microservices Patterns* (Manning)
- Vaughn Vernon, *Implementing Domain-Driven Design* (Addison-Wesley)
- Martin Fowler's Microservices Resource Guide: <https://martinfowler.com/microservices/>
- Microsoft Azure Architecture Center — Microservices: <https://learn.microsoft.com/en-us/azure/architecture/microservices/>
