# Spring Boot Security Rules

일반적인 보안 원칙(입력 검증, 인증/인가, CORS, 보안 헤더, Rate Limiting, 응답 필터링, 시크릿 관리, 안티패턴)은 `security.md` 참조.

이 문서는 Spring Boot에서의 보안 구현 패턴만 다룹니다.

## 1. Bean Validation

```kotlin
data class CreateUserRequest(
    @field:NotBlank(message = "Name is required")
    @field:Size(min = 2, max = 50, message = "Name must be 2-50 characters")
    val name: String,

    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Invalid email format")
    val email: String,

    @field:NotBlank(message = "Password is required")
    @field:Pattern(
        regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@#$%^&+=!]).{8,}$",
        message = "Password must be 8+ chars with upper, lower, digit, special char"
    )
    val password: String
)
```

---

## 2. Spring Security Configuration

```kotlin
@Configuration
@EnableWebSecurity
class SecurityConfig(
    private val jwtAuthFilter: JwtAuthenticationFilter
) {
    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain =
        http
            .csrf { it.disable() } // Disabled for stateless API
            .sessionManagement { it.sessionCreationPolicy(STATELESS) }
            .authorizeHttpRequests {
                it
                    .requestMatchers("/actuator/health/**").permitAll()
                    .requestMatchers("/api/v1/auth/**").permitAll()
                    .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated()
            }
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter::class.java)
            .build()
}
```

---

## 3. Spring CORS Configuration

```kotlin
@Configuration
class CorsConfig {
    @Bean
    fun corsConfigurationSource(): CorsConfigurationSource {
        val config = CorsConfiguration().apply {
            allowedOrigins = listOf("https://app.example.com")
            allowedMethods = listOf("GET", "POST", "PUT", "PATCH", "DELETE")
            allowedHeaders = listOf("Authorization", "Content-Type")
            allowCredentials = true
            maxAge = 3600
        }
        return UrlBasedCorsConfigurationSource().apply {
            registerCorsConfiguration("/api/**", config)
        }
    }
}
```

---

## 4. Rate Limiting with Spring Filter

```kotlin
@Component
class RateLimitFilter(
    private val rateLimiter: RateLimiterService
) : OncePerRequestFilter() {

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val clientId = extractClientId(request)
        val result = rateLimiter.tryConsume(clientId)

        response.setHeader("X-RateLimit-Limit", result.limit.toString())
        response.setHeader("X-RateLimit-Remaining", result.remaining.toString())
        response.setHeader("X-RateLimit-Reset", result.resetAt.toString())

        if (!result.allowed) {
            response.status = HttpStatus.TOO_MANY_REQUESTS.value()
            return
        }

        filterChain.doFilter(request, response)
    }
}
```

---

## 5. Response DTO Pattern

```kotlin
data class UserResponse(
    val id: Long,
    val name: String,
    val email: String,
    val status: String
    // No password, no internal fields
)

fun User.toResponse() = UserResponse(
    id = id,
    name = name,
    email = email,
    status = status.name
)
```

---

## 6. Spring Secret Management

```yaml
# application.yml — no secrets here
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

jwt:
  secret: ${JWT_SECRET}
  expiration: ${JWT_EXPIRATION:3600}
```

### Secret Configuration Rules

- Use `${ENV_VAR}` without defaults for secrets — fail fast if missing
- Use `${ENV_VAR:default}` only for non-secret configuration values
- Keep secrets out of all YAML files including profile-specific ones
