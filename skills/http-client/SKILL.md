---
name: http-client
description: >-
  Framework-agnostic external API integration patterns including timeouts,
  error handling, retry strategy, circuit breaker, and response mapping.
  Use when writing or reviewing HTTP client code.
---
# External API Client Rules

## 1. Timeout Configuration Principles

### Recommended Defaults

| Setting          | Default | Description                |
| ---------------- | ------- | -------------------------- |
| Connect timeout  | 3s      | TCP connection setup       |
| Read timeout     | 10s     | Waiting for response body  |
| Write timeout    | 10s     | Sending request body       |
| Connection pool  | 20      | Max concurrent connections |
| Idle timeout     | 60s     | Idle connection lifetime   |

### Timeout Rules

- Always set explicit timeouts — never rely on defaults (which may be infinite)
- Set read timeout based on the slowest expected response from upstream
- Use shorter timeouts for non-critical calls
- Log timeout values at startup for debugging

---

## 2. Error Handling Strategy

### Error Classification

| Exception Source        | Cause               | Action                 |
| ----------------------- | ------------------- | ---------------------- |
| HTTP response error     | 4xx/5xx response    | Map to domain error    |
| Network/timeout error   | Connection failure  | Retry or circuit break |
| Parsing/decoding error  | Malformed response  | Log and fail           |

### Error Handling Rules

- Never let raw HTTP client exceptions propagate to callers
- Wrap in domain-specific exceptions (e.g., `PaymentApiException`)
- Log response status and body on errors (but mask sensitive data)
- Distinguish between retryable (network, 503) and non-retryable (400, 404) errors

---

## 3. Retry Strategy

### Retry Decision Table

| Scenario           | Retry | Max Attempts | Backoff              |
| ------------------ | ----- | ------------ | -------------------- |
| Network timeout    | Yes   | 3            | Exponential 2x       |
| Connection refused | Yes   | 3            | Exponential 2x       |
| HTTP 429           | Yes   | 3            | Respect Retry-After  |
| HTTP 503           | Yes   | 3            | Exponential 2x       |
| HTTP 4xx (not 429) | No    | -            | -                    |
| Parsing error      | No    | -            | -                    |

### Retry Principles

- Always use exponential backoff with jitter to avoid thundering herd
- Set a maximum retry count — never retry indefinitely
- Do not retry inside a database transaction — retry at the outermost layer
- Do not retry non-idempotent requests without an idempotency key

---

## 4. Circuit Breaker Pattern

### When to Use

- External APIs with unpredictable latency or availability
- Non-critical dependencies where degraded operation is acceptable
- High-throughput paths where cascading failures are a risk

### Key Parameters

| Parameter                    | Description                              | Typical Value |
| ---------------------------- | ---------------------------------------- | ------------- |
| Sliding window size          | Number of calls to evaluate              | 10            |
| Failure rate threshold       | Percentage to open circuit               | 50%           |
| Wait duration in open state  | Cooldown before half-open                | 30s           |
| Half-open permitted calls    | Test calls before closing                | 3             |
| Slow call duration threshold | What counts as "slow"                    | 5s            |
| Slow call rate threshold     | Percentage of slow calls to open circuit | 80%           |

---

## 5. Response Mapping Principles

### DTO Separation

- Never expose external API DTOs to internal service layers
- Map external responses to domain models at the client boundary
- Handle missing/null fields defensively in external DTOs

### Mapping Example

```text
External API Response (their naming)     Domain Model (our naming)
─────────────────────────────────────    ──────────────────────────
user_id: String                    →     id: String
full_name: String                  →     name: String
email_address: String              →     email: String
```

---

## 6. Anti-Patterns

- No timeout configuration (risk of thread exhaustion)
- Retrying non-idempotent requests (POST without idempotency key)
- Logging full response bodies in production (performance + data leak)
- Sharing a single HTTP client instance for unrelated APIs
- Calling external APIs inside database transactions
- Ignoring rate limit headers from upstream APIs
- No circuit breaker on critical external dependencies
