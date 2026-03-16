# Spring Boot API Client Rules

일반적인 외부 API 클라이언트 원칙(타임아웃 전략, 에러 분류, 재시도 전략, Circuit Breaker 파라미터, 응답 매핑 원칙, 안티패턴)은 `http-client.md` 참조.

이 문서는 Spring Boot에서의 구현 패턴만 다룹니다.

## 1. RestClient Setup (Spring Boot 3.2+)

```kotlin
@Configuration
class RestClientConfig(
    private val props: HttpClientProperties
) {
    @Bean
    fun externalApiClient(): RestClient =
        RestClient.builder()
            .baseUrl(props.baseUrl)
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .requestFactory(clientHttpRequestFactory())
            .build()

    private fun clientHttpRequestFactory(): ClientHttpRequestFactory {
        val factory = SimpleClientHttpRequestFactory()
        factory.setConnectTimeout(props.connectTimeout)
        factory.setReadTimeout(props.readTimeout)
        return factory
    }
}
```

---

## 2. RestClient Usage Pattern

```kotlin
@Component
class PaymentApiClient(
    private val restClient: RestClient
) {
    fun getPayment(paymentId: String): PaymentResponse =
        restClient.get()
            .uri("/payments/{id}", paymentId)
            .retrieve()
            .body(PaymentResponse::class.java)
            ?: throw ExternalApiException("Empty response for payment: $paymentId")

    fun createPayment(request: PaymentRequest): PaymentResponse =
        restClient.post()
            .uri("/payments")
            .body(request)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError) { _, response ->
                throw PaymentValidationException(response.statusCode)
            }
            .body(PaymentResponse::class.java)
            ?: throw ExternalApiException("Empty response for create payment")
}
```

---

## 3. Configuration Properties

```kotlin
@ConfigurationProperties(prefix = "app.http-client")
data class HttpClientProperties(
    val baseUrl: String,
    val connectTimeout: Duration = Duration.ofSeconds(3),
    val readTimeout: Duration = Duration.ofSeconds(10),
    val maxConnections: Int = 20
)
```

---

## 4. Spring Retry Configuration

```kotlin
@Configuration
@EnableRetry
class RetryConfig

@Service
class ExternalService(
    private val restClient: RestClient
) {
    @Retryable(
        retryFor = [ResourceAccessException::class],
        noRetryFor = [RestClientResponseException::class],
        maxAttempts = 3,
        backoff = Backoff(delay = 1000, multiplier = 2.0)
    )
    fun fetchData(): DataResponse {
        return restClient.get()
            .uri("/data")
            .retrieve()
            .body(DataResponse::class.java)
            ?: throw ExternalApiException("Empty response")
    }

    @Recover
    fun fetchDataFallback(e: ResourceAccessException): DataResponse {
        log.warn("All retries exhausted for fetchData", e)
        throw ExternalApiException("External API unavailable after retries", e)
    }
}
```

---

## 5. Resilience4j Circuit Breaker

### YAML Configuration

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentApi:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
        slow-call-duration-threshold: 5s
        slow-call-rate-threshold: 80
```

### Annotation Usage

```kotlin
@Service
class PaymentService(
    private val paymentClient: PaymentApiClient
) {
    @CircuitBreaker(name = "paymentApi", fallbackMethod = "paymentFallback")
    fun processPayment(request: PaymentRequest): PaymentResponse {
        return paymentClient.createPayment(request)
    }

    fun paymentFallback(request: PaymentRequest, e: Exception): PaymentResponse {
        log.warn("Circuit breaker open for payment API", e)
        throw PaymentUnavailableException("Payment service temporarily unavailable")
    }
}
```

---

## 6. Response DTO Mapping

```kotlin
// External API response (matches their contract)
data class ExternalUserResponse(
    val user_id: String,
    val full_name: String,
    val email_address: String
)

// Domain model (our naming convention)
data class User(
    val id: String,
    val name: String,
    val email: String
)

// Mapper extension function
fun ExternalUserResponse.toDomain() = User(
    id = user_id,
    name = full_name,
    email = email_address
)
```
