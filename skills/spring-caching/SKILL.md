# Spring Boot Caching Rules

일반적인 캐싱 원칙(캐시 전략 선택, TTL 설계, 무효화 패턴, 안티패턴)은 `caching.md` 참조.

이 문서는 Spring Boot에서의 캐시 구현 패턴만 다룹니다.

## 1. Spring Cache Abstraction

### Setup

```kotlin
@Configuration
@EnableCaching
class CacheConfig
```

### Usage

```kotlin
@Service
class UserService(
    private val userRepository: UserRepository
) {
    @Cacheable(value = ["users"], key = "#id")
    fun findById(id: Long): User =
        userRepository.findByIdOrThrow(id)

    @CachePut(value = ["users"], key = "#user.id")
    fun update(user: User): User =
        userRepository.save(user)

    @CacheEvict(value = ["users"], key = "#id")
    fun delete(id: Long) =
        userRepository.deleteById(id)

    @CacheEvict(value = ["users"], allEntries = true)
    fun refreshAll() {
        log.info("All user cache entries evicted")
    }
}
```

### Annotation Reference

| Annotation     | Purpose                         | When Called        |
| -------------- | ------------------------------- | ------------------ |
| `@Cacheable`   | Return cached value if exists   | Before method      |
| `@CachePut`    | Update cache with return value  | After method       |
| `@CacheEvict`  | Remove entry from cache         | After method       |
| `@Caching`     | Combine multiple cache ops      | Composite          |

---

## 2. Caffeine Local Cache

### Default Configuration

```kotlin
@Configuration
@EnableCaching
class CaffeineCacheConfig {

    @Bean
    fun cacheManager(): CacheManager =
        CaffeineCacheManager().apply {
            setCaffeine(
                Caffeine.newBuilder()
                    .maximumSize(1000)
                    .expireAfterWrite(Duration.ofMinutes(10))
                    .recordStats()
            )
        }
}
```

### Per-Cache Configuration

```kotlin
@Bean
fun cacheManager(): CacheManager =
    SimpleCacheManager().apply {
        setCaches(listOf(
            CaffeineCache("users",
                Caffeine.newBuilder()
                    .maximumSize(500)
                    .expireAfterWrite(Duration.ofMinutes(30))
                    .build()),
            CaffeineCache("config",
                Caffeine.newBuilder()
                    .maximumSize(100)
                    .expireAfterWrite(Duration.ofHours(1))
                    .build())
        ))
    }
```

### When to Use Caffeine

- Single-instance applications
- Hot data with high read frequency
- Data that can tolerate per-instance inconsistency
- As L1 cache in front of Redis (two-level caching)

---

## 3. Redis Distributed Cache

### Spring Boot Configuration

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
      timeout: ${REDIS_TIMEOUT:3s}
```

### Cache Manager

```kotlin
@Configuration
@EnableCaching
class RedisCacheConfig {

    @Bean
    fun cacheManager(connectionFactory: RedisConnectionFactory): CacheManager =
        RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig())
            .withCacheConfiguration("users", cacheConfig(Duration.ofMinutes(30)))
            .withCacheConfiguration("config", cacheConfig(Duration.ofHours(1)))
            .build()

    private fun defaultConfig() = cacheConfig(Duration.ofMinutes(10))

    private fun cacheConfig(ttl: Duration) =
        RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(ttl)
            .serializeKeysWith(
                SerializationPair.fromSerializer(StringRedisSerializer()))
            .serializeValuesWith(
                SerializationPair.fromSerializer(GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues()
}
```

### When to Use Redis

- Multi-instance deployments (cache must be shared)
- Session storage
- Rate limiting counters
- Data that must survive application restarts

---

## 4. Event-Based Invalidation

```kotlin
@Component
class CacheInvalidationListener(
    private val cacheManager: CacheManager
) {
    @EventListener
    fun onUserUpdated(event: UserUpdatedEvent) {
        cacheManager.getCache("users")?.evict(event.userId)
    }
}
```
