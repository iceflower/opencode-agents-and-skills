# Spring Boot Error Handling Rules

일반적인 에러 처리 원칙(예외 계층, 에러 응답 포맷, 계층별 가이드라인, 외부 API 에러 처리, 안티패턴)은 `error-handling.md` 참조.

이 문서는 Spring Boot에서의 구현 패턴만 다룹니다.

## 1. Business Exception Design

### Base Exception

```kotlin
abstract class BusinessException(
    val errorCode: ErrorCode,
    override val message: String = errorCode.message
) : RuntimeException(message)
```

### Specific Exceptions

```kotlin
class EntityNotFoundException(
    entityName: String,
    id: Any
) : BusinessException(
    ErrorCode.ENTITY_NOT_FOUND,
    "$entityName not found: $id"
)

class DuplicateEntityException(
    field: String,
    value: Any
) : BusinessException(
    ErrorCode.DUPLICATE_ENTITY,
    "Duplicate $field: $value"
)
```

---

## 2. Error Code Enum

```kotlin
enum class ErrorCode(
    val status: HttpStatus,
    val code: String,
    val message: String
) {
    // Common
    INVALID_INPUT(HttpStatus.BAD_REQUEST, "C001", "Invalid input"),
    ENTITY_NOT_FOUND(HttpStatus.NOT_FOUND, "C002", "Entity not found"),
    DUPLICATE_ENTITY(HttpStatus.CONFLICT, "C003", "Duplicate entity"),

    // Auth
    UNAUTHORIZED(HttpStatus.UNAUTHORIZED, "A001", "Authentication required"),
    ACCESS_DENIED(HttpStatus.FORBIDDEN, "A002", "Access denied"),

    // Internal
    INTERNAL_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "S001", "Internal server error"),
    EXTERNAL_API_ERROR(HttpStatus.BAD_GATEWAY, "S002", "External API error"),
}
```

---

## 3. @ControllerAdvice Pattern

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    // Business exceptions → 4xx
    @ExceptionHandler(BusinessException::class)
    fun handleBusinessException(e: BusinessException): ResponseEntity<ErrorResponse> {
        log.warn("Business exception: ${e.errorCode.code}", e)
        return ResponseEntity
            .status(e.errorCode.status)
            .body(ErrorResponse.of(e.errorCode, e.message))
    }

    // Validation errors → 400
    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(e: MethodArgumentNotValidException): ResponseEntity<ErrorResponse> {
        val details = e.bindingResult.fieldErrors.map {
            ErrorResponse.FieldError(it.field, it.defaultMessage ?: "Invalid value")
        }
        return ResponseEntity
            .badRequest()
            .body(ErrorResponse.of(ErrorCode.INVALID_INPUT, details = details))
    }

    // Uncaught exceptions → 500
    @ExceptionHandler(Exception::class)
    fun handleException(e: Exception): ResponseEntity<ErrorResponse> {
        log.error("Unhandled exception", e)
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse.of(ErrorCode.INTERNAL_ERROR))
    }
}
```

---

## 4. Error Response Implementation

```kotlin
data class ErrorResponse(
    val error: ErrorDetail,
    val meta: Meta
) {
    data class ErrorDetail(
        val code: String,
        val message: String,
        val details: List<FieldError> = emptyList()
    )

    data class FieldError(
        val field: String,
        val message: String
    )

    data class Meta(
        val timestamp: Instant = Instant.now(),
        val requestId: String? = MDC.get("traceId")
    )

    companion object {
        fun of(errorCode: ErrorCode, message: String? = null, details: List<FieldError> = emptyList()) =
            ErrorResponse(
                error = ErrorDetail(errorCode.code, message ?: errorCode.message, details),
                meta = Meta()
            )
    }
}
```

---

## 5. External API Error Handling with RestClient

```kotlin
fun callExternalApi(): ExternalResponse {
    return try {
        restClient.get()
            .uri("/external/resource")
            .retrieve()
            .body(ExternalResponse::class.java)
            ?: throw ExternalApiException("Empty response from external API")
    } catch (e: RestClientException) {
        log.error("External API call failed", e)
        throw ExternalApiException("External API unavailable", e)
    }
}
```

### Spring Retry Strategy

- Use Spring Retry (`@Retryable`) for transient failures
- Set max attempts (typically 3) and backoff (exponential)
- Define `@Recover` method for final fallback
