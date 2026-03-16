---
name: testing-integration
description: >-
  Integration testing patterns including test containers, mock servers,
  test data management, and test organization.
  Use when writing integration tests with real dependencies.
---
# Integration Testing Rules

## 1. Integration vs Unit Test Boundaries

| Aspect          | Unit Test                    | Integration Test                    |
| --------------- | ---------------------------- | ----------------------------------- |
| Scope           | Single class/function        | Multiple components or external I/O |
| Dependencies    | Mocked or faked              | Real or containerized               |
| Speed           | Milliseconds                 | Seconds to minutes                  |
| When to use     | Business logic, algorithms   | DB queries, API calls, workflows    |
| Isolation       | Full                         | Partial (shared test DB, etc.)      |

### When to Write Integration Tests

- Database queries and repository methods
- External API client behavior (with mock server)
- End-to-end request/response through controllers
- Message producer/consumer workflows
- Cache behavior with real cache instance

---

## 2. Test Container Strategy

### Recommended Containers

| Dependency   | Container                      | Use Case                    |
| ------------ | ------------------------------ | --------------------------- |
| PostgreSQL   | `postgres:16-alpine`           | Database integration tests  |
| MySQL        | `mysql:8`                      | Database integration tests  |
| Redis        | `redis:7-alpine`               | Cache integration tests     |
| Kafka        | `confluentinc/cp-kafka`        | Messaging integration tests |
| LocalStack   | `localstack/localstack`        | AWS service mocking         |
| WireMock     | `wiremock/wiremock`            | External API mocking        |

### Container Rules

- Use fixed image tags — never `latest` in tests
- Share containers across test classes for speed (singleton pattern)
- Clean up test data between tests, not containers
- Use `@DynamicPropertySource` or equivalent to inject container ports

---

## 3. Test Data Management

### Setup Strategies

| Strategy           | Pros               | Cons                     |
| ------------------ | ------------------ | ------------------------ |
| Per-test setup     | Full isolation     | Slower, more boilerplate |
| Shared fixtures    | Faster, less code  | Risk of test coupling    |
| Database migration | Realistic schema   | Slower initial setup     |
| SQL scripts        | Explicit, readable | Maintenance burden       |

### Data Management Rules

- Each test must be independent — never rely on execution order
- Clean up test data after each test (truncate tables or use transactions)
- Use meaningful test data that reflects real scenarios
- Never use production data in tests — generate synthetic data
- Avoid shared mutable state between test classes

---

## 4. Mock Server Patterns

### External API Mocking

- Use WireMock or MockServer for HTTP dependency mocking
- Define stubs that match the real API contract
- Test both success and error responses from external APIs
- Verify request format sent to external APIs (headers, body, query params)

### Mock Server Rules

- Keep mock responses as close to real API responses as possible
- Test timeout scenarios (delayed responses)
- Test rate limiting responses (429)
- Test server error responses (500, 503)
- Reset mock server state between tests

---

## 5. Test Organization

### Naming Convention

```text
{TargetClass}IntegrationTest
{Feature}IntegrationTest
{Endpoint}ApiTest
```

### Structure

- Separate integration tests from unit tests (different source set or directory)
- Use test profiles with dedicated configuration
- Tag integration tests for selective execution in CI
- Keep integration tests focused — test one integration point per class

---

## 6. Performance Considerations

- Reuse containers across test suites (singleton pattern)
- Use parallel test execution only when tests are truly independent
- Minimize test data volume — use only what the test needs
- Cache container images in CI for faster startup
- Set appropriate timeouts for container startup and test execution

---

## 7. Anti-Patterns

- Testing everything through integration tests (use unit tests for logic)
- Starting new containers for each test class
- Hardcoded ports or connection strings
- Tests that depend on execution order
- No cleanup between tests (data leaks between test cases)
- Mocking everything in integration tests (defeats the purpose)
- Missing error scenario coverage (only testing happy paths)
