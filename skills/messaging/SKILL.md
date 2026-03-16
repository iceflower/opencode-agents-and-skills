---
name: messaging
description: >-
  Async messaging patterns including broker selection, message design,
  producer/consumer patterns, schema evolution, and monitoring.
  Use when implementing Kafka, RabbitMQ, or event-driven architecture.
---
# Messaging Rules

## 1. Message Broker Selection

| Broker           | Strengths                                      | Best For                               |
| ---------------- | ---------------------------------------------- | -------------------------------------- |
| Kafka            | High throughput, durability, ordering          | Event streaming, log aggregation       |
| RabbitMQ         | Flexible routing, low latency                  | Task queues, request-reply             |
| NATS Core        | Ultra-low latency, lightweight, simple         | Microservice RPC, IoT, edge computing  |
| NATS JetStream   | Persistence on NATS, exactly-once, KV/Object   | Durable streaming with NATS simplicity |
| Apache Pulsar    | Multi-tenancy, tiered storage, geo-replication | Multi-tenant SaaS, hybrid cloud        |
| Redis Pub/Sub    | Simple, fast                                   | Real-time notifications, caching       |

### Selection Criteria

- Use **Kafka** when ordering, replay, and ecosystem maturity are required
- Use **RabbitMQ** when flexible routing and acknowledgment patterns are priorities
- Use **NATS Core** when ultra-low latency and operational simplicity are critical (no persistence needed)
- Use **NATS JetStream** when you need Kafka-like durability with NATS operational simplicity
- Use **Apache Pulsar** when multi-tenancy, tiered storage, or geo-replication are requirements
- Use **Redis Pub/Sub** only for ephemeral messages where loss is acceptable

### Broker Comparison Matrix

| Feature                | Kafka         | RabbitMQ          | NATS Core  | NATS JetStream | Pulsar        | Redis Pub/Sub |
| ---------------------- | ------------- | ----------------- | ---------- | -------------- | ------------- | ------------- |
| Persistence            | Yes           | Yes               | No         | Yes            | Yes           | No            |
| Ordering guarantee     | Per-partition | Per-queue         | No         | Per-stream     | Per-partition | No            |
| Message replay         | Yes           | Limited           | No         | Yes            | Yes           | No            |
| Exactly-once           | Yes           | No                | No         | Yes            | Yes           | No            |
| Multi-tenancy          | Limited       | Vhost             | Account    | Account        | Native        | No            |
| Geo-replication        | MirrorMaker   | Shovel/Federation | Leaf nodes | Leaf nodes     | Native        | No            |
| Operational complexity | High          | Medium            | Low        | Low-Medium     | High          | Low           |
| Library maturity       | Excellent     | Excellent         | Good       | Good           | Good          | Excellent     |
| Throughput             | Very high     | Medium            | Very high  | High           | Very high     | High          |
| Latency                | Medium        | Low               | Ultra-low  | Low            | Medium        | Ultra-low     |

---

## 2. Message Design

### Message Structure

```json
{
  "messageId": "uuid-v4",
  "type": "order.created",
  "source": "order-service",
  "timestamp": "2024-01-15T10:30:45.123Z",
  "version": "1.0",
  "data": {
    "orderId": "order-123",
    "userId": "user-456",
    "totalAmount": 15000
  },
  "metadata": {
    "traceId": "abc-123",
    "correlationId": "req-789"
  }
}
```

### Message Design Rules

- Always include a unique `messageId` for deduplication
- Include `traceId` for distributed tracing correlation
- Use a `version` field to support schema evolution
- Keep messages self-contained — consumers should not need to call back to the producer
- Use past tense for event names (`order.created`, not `create.order`)

---

## 3. Producer Patterns

### Delivery Guarantees

| Guarantee      | Description                        | Trade-off              |
| -------------- | ---------------------------------- | ---------------------- |
| At-most-once   | Fire and forget                    | May lose messages      |
| At-least-once  | Retry until acknowledged           | May duplicate messages |
| Exactly-once   | Deduplication + transactional send | Highest complexity     |

### Producer Rules

- Default to at-least-once delivery — it is the safest general-purpose guarantee
- Use transactional outbox pattern for database + message atomicity
- Never publish messages inside a database transaction without outbox
- Include idempotency keys so consumers can deduplicate

---

## 4. Consumer Patterns

### Idempotent Consumers

- Always design consumers to handle duplicate messages
- Use `messageId` or business key for deduplication
- Store processed message IDs with TTL to prevent reprocessing

### Error Handling

| Strategy           | When to Use                          | Implementation                 |
| ------------------ | ------------------------------------ | ------------------------------ |
| Retry with backoff | Transient errors (network, timeout)  | Exponential backoff, max 3-5   |
| Dead letter queue  | Persistent failures                  | Route to DLQ after max retries |
| Skip and log       | Poison messages (invalid format)     | Log error, acknowledge message |

### Consumer Rules

- Always set a maximum retry count — never retry indefinitely
- Route failed messages to a dead letter queue (DLQ) for investigation
- Monitor DLQ size — growing DLQ indicates unresolved issues
- Process messages in order only when business logic requires it

---

## 5. NATS-Specific Patterns

### NATS Core

- Use subject-based addressing (`orders.created`, `payments.>` wildcard)
- Leverage request-reply pattern for synchronous microservice communication
- Use queue groups for load balancing across consumer instances
- NATS Core is fire-and-forget — no persistence or replay

### NATS JetStream

- Use streams for durable message storage with configurable retention
- Leverage consumer groups with durable names for reliable processing
- Use Key-Value store for lightweight configuration or state sharing
- Use Object store for large payload handling (claim check pattern)
- Configure stream limits: `max_msgs`, `max_bytes`, `max_age` to prevent unbounded growth
- Prefer pull-based consumers over push-based for backpressure control

### NATS Design Rules

- Use dot-separated hierarchical subjects (`service.entity.action`)
- Leverage wildcards (`*` single token, `>` multi-token) for flexible subscriptions
- Keep payloads small (< 1MB for Core, configurable for JetStream)
- Use headers for metadata (traceId, version) instead of embedding in payload

---

## 6. Pulsar-Specific Patterns

### Topic and Namespace Design

- Use tenant/namespace/topic hierarchy (`public/orders/created`)
- Leverage namespaces for access control and policy isolation
- Use partitioned topics for high-throughput scenarios
- Configure topic-level policies (retention, TTL, backlog quota) per use case

### Subscription Modes

| Mode       | Behavior                              | Use Case                          |
| ---------- | ------------------------------------- | --------------------------------- |
| Exclusive  | Single consumer per subscription      | Ordered processing                |
| Failover   | Active-standby consumer pair          | High availability                 |
| Shared     | Round-robin across consumers          | Parallel processing               |
| Key_Shared | Partition by key across consumers     | Ordered per-key parallel          |

### Pulsar Design Rules

- Use tiered storage for cost-effective long-term retention
- Leverage schema registry with Avro/Protobuf for type safety
- Use delayed message delivery for scheduled tasks
- Configure backlog quota policies to prevent unbounded topic growth
- Use geo-replication for multi-region disaster recovery

---

## 7. Schema Evolution (All Brokers)

### Backward Compatibility

- Adding new optional fields is always safe
- Removing fields or changing types is a breaking change
- Use schema registry (Avro, Protobuf) for strict contract enforcement
- Version your message schemas and support at least N-1 version

### Migration Strategy

- Deploy consumers that support both old and new schema first
- Then deploy producers with the new schema
- Never deploy producer changes before consumer compatibility is verified

---

## 8. Monitoring

### Key Metrics

| Metric                                 | Alert Threshold   | Severity |
| -------------------------------------- | ----------------- | -------- |
| Consumer lag                           | Growing over time | Warning  |
| DLQ message count                      | > 0               | Warning  |
| Message processing time                | > SLA threshold   | Warning  |
| Producer error rate                    | > 1%              | Critical |
| Consumer group rebalance frequency     | Frequent          | Warning  |

---

## 9. Anti-Patterns

- Publishing messages inside database transactions (use outbox pattern)
- No dead letter queue for failed messages
- Consumers that assume message ordering without partition keys
- Missing idempotency handling in consumers
- Unbounded retry without backoff or max attempts
- Large message payloads (> 1MB) — use claim check pattern instead
- No schema versioning for message contracts
