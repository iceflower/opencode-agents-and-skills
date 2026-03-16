# Spring Boot Monitoring Rules

일반적인 모니터링 원칙(옵저버빌리티 3대 요소, 메트릭 유형/명명, 핵심 메트릭, 알림 규칙, 헬스체크 개념, 안티패턴)은 `monitoring.md` 참조.

이 문서는 Spring Boot에서의 모니터링 구현 패턴만 다룹니다.

## 1. Spring Boot Actuator Setup

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
    tags:
      application: ${spring.application.name}
```

### Actuator Security Rules

- Never expose all actuator endpoints in production
- Protect sensitive endpoints (env, configprops, beans) behind authentication
- Use separate management port for internal-only endpoints when needed

---

## 2. Micrometer Custom Metrics

```kotlin
@Component
class OrderMetrics(
    private val meterRegistry: MeterRegistry
) {
    private val orderCounter = Counter.builder("orders.created")
        .description("Number of orders created")
        .register(meterRegistry)

    private val orderTimer = Timer.builder("orders.processing.duration")
        .description("Order processing duration")
        .register(meterRegistry)

    fun recordOrderCreated() {
        orderCounter.increment()
    }

    fun <T> timeOrderProcessing(block: () -> T): T {
        return orderTimer.recordCallable(block)!!
    }
}
```

---

## 3. Distributed Tracing Setup

### Configuration

```yaml
management:
  tracing:
    sampling:
      probability: 1.0  # 100% in dev, lower in prod

logging:
  pattern:
    correlation: "[${spring.application.name:},%mdc{traceId:-},%mdc{spanId:-}] "
```

### Dependencies

```kotlin
// build.gradle.kts
implementation("io.micrometer:micrometer-tracing-bridge-otel")
implementation("io.opentelemetry:opentelemetry-exporter-otlp")
```

### Spring Propagation Rules

- TraceId propagates automatically through Spring WebMVC, RestClient, WebClient
- For async operations, use `@Async` with `TaskDecorator` to propagate context
- For messaging (Kafka, RabbitMQ), use Micrometer observation wrappers
- Always include traceId in log patterns for correlation

---

## 4. Custom Spans with Observation API

```kotlin
@Component
class PaymentProcessor(
    private val observationRegistry: ObservationRegistry
) {
    fun processPayment(orderId: String): PaymentResult {
        val observation = Observation.createNotStarted("payment.process", observationRegistry)
            .lowCardinalityKeyValue("payment.method", "card")
            .highCardinalityKeyValue("order.id", orderId)

        return observation.observe {
            callPaymentGateway(orderId)
        }!!
    }
}
```

---

## 5. Spring Health Check Implementation

```kotlin
@Component
class ExternalApiHealthIndicator(
    private val restClient: RestClient
) : HealthIndicator {

    override fun health(): Health {
        return try {
            restClient.get()
                .uri("/health")
                .retrieve()
                .toBodilessEntity()
            Health.up()
                .withDetail("api", "reachable")
                .build()
        } catch (e: Exception) {
            Health.down(e)
                .withDetail("api", "unreachable")
                .build()
        }
    }
}
```
