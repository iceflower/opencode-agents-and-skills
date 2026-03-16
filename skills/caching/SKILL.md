---
name: caching
description: >-
  Framework-agnostic caching strategies including cache selection, TTL design,
  invalidation patterns, and anti-patterns.
  Use when designing or reviewing cache logic.
---

# Caching Rules

## 1. Cache Strategy Selection

### When to Cache

| Scenario                              | Cache? | Reason                          |
| ------------------------------------- | ------ | ------------------------------- |
| Frequently read, rarely written data  | Yes    | High read-to-write ratio        |
| Expensive computation results         | Yes    | Reduce CPU/DB load              |
| External API responses                | Yes    | Reduce latency and rate limits  |
| User session data                     | Yes    | Avoid repeated DB lookups       |
| Real-time data (stock prices, etc.)   | No     | Stale data is unacceptable      |
| Write-heavy data                      | No     | Cache invalidation too frequent |
| Security-critical data (permissions)  | Careful| Must invalidate on change       |

### Cache Placement

| Type          | Latency | Scope           |
| ------------- | ------- | --------------- |
| Local (L1)    | ~1ms    | Single instance |
| Distributed   | ~5ms    | All instances   |
| HTTP cache    | Varies  | Client/CDN      |

---

## 2. TTL Design Guidelines

### Recommended TTL by Data Type

| Data Type               | Recommended TTL  | Rationale                   |
| ----------------------- | ---------------- | --------------------------- |
| Static config           | 1-24 hours       | Rarely changes              |
| User profile            | 15-30 minutes    | Moderate change frequency   |
| Search results          | 5-10 minutes     | Freshness matters           |
| API rate limit counters | 1 minute window  | Must be accurate            |
| Session data            | 30 min - 2 hours | Balance UX and security     |

### TTL Rules

- Always set a TTL — never cache indefinitely
- Shorter TTL for data that affects user experience when stale
- Longer TTL for reference data that changes infrequently
- Use explicit eviction on writes in addition to TTL
- Consider TTL jitter to avoid cache stampede

---

## 3. Cache Invalidation Patterns

### Write-Through

Update the cache entry simultaneously when writing to the data store. Ensures cache is always consistent but adds write latency.

### Evict on Write

Evict the cache entry on write and let the next read repopulate it. Simpler to implement and avoids stale data from failed cache updates.

### Event-Based Invalidation

Invalidate cache entries via events (message queue, application events) for cross-service consistency. Best for distributed systems where multiple services share cached data.

### Invalidation Rules

- Prefer explicit eviction over TTL-only for mutable data
- Invalidate on all write paths (update, delete, bulk operations)
- Use event-based invalidation for cross-service cache consistency
- Test cache invalidation paths — stale cache bugs are hard to diagnose

---

## 4. Anti-Patterns

- Caching without TTL (memory leak risk)
- Caching mutable objects (caller modifies cached reference)
- Cache key collisions (using only entity ID without type prefix)
- Caching null values without explicit handling
- Ignoring cache in delete/update paths (stale reads)
- Over-caching (caching everything "just in case")
- No monitoring of cache hit/miss rates
- Using distributed cache for data that only needs local caching
