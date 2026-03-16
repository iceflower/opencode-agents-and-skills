# REST API Design Rules

## 1. URL Design

### Basic Principles

- Use **nouns**, not verbs, to represent resources
- Use **plural forms** instead of singular
- Use **kebab-case** for URL paths (lowercase with hyphens)
- Represent **hierarchical relationships** in the URL structure

### URL Patterns

```text
# Resource collection
GET    /users                  # List users
POST   /users                  # Create user

# Specific resource
GET    /users/{id}             # Get specific user
PUT    /users/{id}             # Full update
PATCH  /users/{id}             # Partial update
DELETE /users/{id}             # Delete user

# Sub-resources
GET    /users/{id}/orders      # List user's orders
POST   /users/{id}/orders      # Create order for user
GET    /users/{id}/orders/{orderId}  # Get specific order

# Actions (when noun representation is difficult)
POST   /users/{id}/password-reset   # Reset password
POST   /orders/{id}/cancel          # Cancel order
```

### Anti-Patterns

```text
# Bad examples
GET    /getUsers
POST   /createUser
DELETE /deleteUser/123
GET    /user              # singular form
GET    /Users             # uppercase
GET    /user_orders       # snake_case

# Good examples
GET    /users
POST   /users
DELETE /users/123
```

---

## 2. HTTP Methods

### Method Usage

| Method  | Purpose         | Idempotent | Safe | Request Body |
| ------- | --------------- | ---------- | ---- | ------------ |
| GET     | Retrieve        | Yes        | Yes  | No           |
| POST    | Create          | No         | No   | Yes          |
| PUT     | Full Update     | Yes        | No   | Yes          |
| PATCH   | Partial Update  | No         | No   | Yes          |
| DELETE  | Remove          | Yes        | No   | No           |

### Idempotency

- **Idempotent**: Multiple identical requests produce the same result
- GET, PUT, DELETE must guarantee idempotency
- POST is not idempotent → duplicate creation prevention logic required

---

## 3. HTTP Status Codes

### Success (2xx)

| Code | Meaning       | Use Case                       |
| ---- | ------------- | ------------------------------ |
| 200  | OK            | General success                |
| 201  | Created       | Resource created successfully  |
| 202  | Accepted      | Async processing started       |
| 204  | No Content    | Success with no response body  |

### Redirection (3xx)

| Code | Meaning             | Use Case                      |
| ---- | ------------------- | ----------------------------- |
| 301  | Moved Permanently   | Resource permanently moved    |
| 302  | Found               | Temporary redirect            |
| 304  | Not Modified        | Cached resource unchanged     |

### Client Errors (4xx)

| Code | Meaning               | Use Case                |
| ---- | --------------------- | ----------------------- |
| 400  | Bad Request           | Invalid request format  |
| 401  | Unauthorized          | Authentication required |
| 403  | Forbidden             | No permission           |
| 404  | Not Found             | Resource not found      |
| 409  | Conflict              | Resource conflict       |
| 422  | Unprocessable Entity  | Validation failed       |
| 429  | Too Many Requests     | Rate limit exceeded     |

### Server Errors (5xx)

| Code | Meaning               | Use Case                  |
| ---- | --------------------- | ------------------------- |
| 500  | Internal Server Error | Server internal error     |
| 502  | Bad Gateway           | Upstream server error     |
| 503  | Service Unavailable   | Service temporarily down  |
| 504  | Gateway Timeout       | Upstream server timeout   |

---

## 4. Request/Response Format

### Request Headers

```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer <token>
X-Request-ID: <uuid>
```

### Response Format - Success

```json
{
  "data": {
    "id": "user-001",
    "email": "user@example.com",
    "name": "John Doe"
  },
  "meta": {
    "timestamp": "2024-01-15T10:30:45.123Z",
    "requestId": "abc-123"
  }
}
```

### Response Format - List

```json
{
  "data": [
    { "id": "user-001", "name": "John Doe" },
    { "id": "user-002", "name": "Jane Doe" }
  ],
  "meta": {
    "total": 100,
    "page": 1,
    "perPage": 20,
    "totalPages": 5
  }
}
```

### Response Format - Error

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ]
  },
  "meta": {
    "timestamp": "2024-01-15T10:30:45.123Z",
    "requestId": "abc-123"
  }
}
```

---

## 5. Pagination

### Pagination Strategy Selection

| Method | Pros                      | Cons                    | Best For          |
| ------ | ------------------------- | ----------------------- | ----------------- |
| Offset | Simple, easy page nav     | Slow on large datasets  | Small datasets    |
| Cursor | Fast, consistent          | No page navigation      | Large datasets    |
| Keyset | Fast                      | Fixed sort key          | Fixed sort order  |

---

## 6. Filtering, Sorting, Field Selection

```text
# Filtering
GET /users?status=active&role=admin

# Sorting
GET /users?sort=-createdAt         # descending
GET /users?sort=name,-createdAt    # multiple sort

# Field Selection
GET /users?fields=id,name,email
```

---

## 7. API Versioning

API는 서비스 간 계약 역할을 하며, 변경 시 기존 클라이언트가 영향을 받지 않도록 신중한 계획이 필요합니다.

### Version Identification Strategies

#### URL Path Versioning

**Method**: Add version identifier prefix or query parameter to URL.

```text
https://api.example.com/v1/users
https://api.example.com/v2/users
https://api.example.com/users?version=1
```

**Pros**:

- Easy route branching
- URLs easy to share, save, email
- Can view version consumption via logs
- Users can see which version they're using at a glance

**Cons**:

- Same resource appears as different resources (serious issue in REST world)

#### HTTP Header Versioning (Accept/Content-Type)

**Method**: Use `Accept` header for requests, `Content-Type` for responses.

```text
Accept: application/vnd.example.user.v1+json
Accept: application/vnd.example.user.v2+json
```

**Pros**:

- All stored URLs remain valid
- Clients can upgrade versions without changing paths

**Cons**:

- Can't identify version from URL alone
- Generic media types like `application/json` or `text/xml` don't help
- Clients must know special media types exist and their allowed range

#### Custom Header Versioning

**Method**: Define application-specific header.

```text
api-version: 1
api-version: 2
```

**Pros**:

- Completely flexible
- Orthogonal to media types and URLs (no interference)

**Cons**:

- Need framework-specific route branching code
- Another piece of secret knowledge to share with service consumers

#### Request Body Field Versioning

**Method**: Add field in request body to indicate intended version (PUT/POST only).

**Pros**:

- Simple for write operations

**Cons**:

- Only works for PUT/POST
- Not applicable for GET requests

### Compatibility Principles

#### Postel's Robustness Principle

> "Be conservative in what you send, be liberal in what you accept."

- **Covariant requests**: Accept more than required
- **Contravariant responses**: Return less than or equal to promised

#### Safe Changes (Always Compatible)

Changes that never break compatibility:

| Safe Change                                         | Example                                              |
| --------------------------------------------------- | ---------------------------------------------------- |
| Require a **subset** of previously required parameters | Made optional parameter required → now optional again |
| Accept a **superset** of previously accepted parameters | Add new optional parameter                          |
| Return a **superset** of previously returned values    | Add new response field                              |
| Require a **subset** of previous constraints           | Loosen validation rules                             |

#### Breaking Changes (Never Compatible)

Changes that always break compatibility:

| Breaking Change                                    | Example                                                 |
| -------------------------------------------------- | ------------------------------------------------------- |
| Reject previously accepted optional information    | Was accepting `nickname`, now rejecting it              |
| Remove previously guaranteed response information  | Was returning `email`, now removing it                  |
| Require higher privilege level                     | Was allowing `user` role, now requiring `admin`         |

### Compatibility Problem Example

A service receives JSON with a URL field:

1. **Initial**: Accepts any string, stores in database without URL validation
2. **Change**: Adds regex-based URL validation
3. **Problem**: Previously valid requests now rejected → **Compatibility broken**

**Solution**:

- Use database migration to fix existing invalid URLs
- Or accept that some historical data doesn't meet new standards
- Version the validation rules if clients depend on old behavior

### Handling Breaking Changes

#### Version Bump Strategy

When breaking changes are unavoidable:

1. **Create new version** (v2) alongside old version (v1)
2. **Both versions coexist** during transition period
3. **Deprecate old version** with clear timeline
4. **Remove old version** after transition period

#### Version Lifecycle

```text
v1 Released → v1 Stable → v2 Released → v1 Deprecated → v1 Sunset → v1 Removed
                                     (6 months)        (3 months)    (EOL)
```

#### Deprecation Notice

Include in API responses:

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 01 Jul 2024 00:00:00 GMT
Link: </v2/users>; rel="successor-version"
```

### Contract Testing

Verify that both provider and consumer sides honor the API contract.

```text
┌─────────────────┐         ┌─────────────────┐
│   Consumer      │         │    Provider     │
│   Test Code     │         │   Test Code     │
└────────┬────────┘         └────────┬────────┘
         │                           │
         ▼                           ▼
┌─────────────────────────────────────────────┐
│              Contract (Pact)                 │
│         - Request expectations              │
│         - Response expectations             │
└─────────────────────────────────────────────┘
```

**Benefits**:

- Test without real external service calls
- Catch breaking changes before deployment
- Verify both sides independently

### Versioning Other Services

#### When You're the Consumer

| Situation                                  | Response                         |
| ------------------------------------------ | -------------------------------- |
| External service announces deprecation     | Plan migration immediately       |
| External service breaks compatibility      | Have fallback strategies         |
| External service unavailable               | Circuit breakers, fallbacks      |

- Document all external service versions used
- Track deprecation timelines
- Maintain version compatibility matrix

### Versioning Checklist

#### Before Making Changes

- [ ] Is this change backward compatible?
- [ ] If breaking, is a new version necessary?
- [ ] Have existing clients been notified?
- [ ] Is there a transition timeline?

#### When Releasing New Version

- [ ] Old version still available during transition?
- [ ] Documentation updated for both versions?
- [ ] Deprecation notices included in responses?
- [ ] Migration guide provided for clients?

#### When Removing Old Version

- [ ] All clients migrated?
- [ ] Sunset date passed?
- [ ] Monitoring shows no traffic to old version?

---

## 8. Security

- All APIs must be served over HTTPS only
- Never include sensitive data in URLs — use request body
- Never expose sensitive data (passwords, hashed values) in responses
- Implement rate limiting with `X-RateLimit-*` headers

---

## 9. Documentation

### OpenAPI Specification Structure

#### Required Sections

```yaml
openapi: 3.1.0
info:
  title: Service Name API
  version: 1.0.0
  description: Brief service description
  contact:
    name: Team Name

servers:
  - url: https://api.example.com/v1
    description: Production

paths:
  /resources:
    get:
      summary: Short action description
      operationId: listResources
      tags:
        - Resources
      parameters: []
      responses: {}

components:
  schemas: {}
  securitySchemes: {}
```

#### General Documentation Rules

- Every endpoint must have a `summary` (short) and optionally a `description` (detailed)
- Use `operationId` for unique identification — must be unique across the spec
- Group related endpoints with `tags`
- Define reusable schemas in `components/schemas`
- Define security schemes in `components/securitySchemes`

### Schema Documentation

#### Request/Response Schema

```yaml
components:
  schemas:
    User:
      type: object
      required:
        - name
        - email
      properties:
        id:
          type: integer
          format: int64
          description: Unique user identifier
          readOnly: true
          example: 42
        name:
          type: string
          minLength: 2
          maxLength: 50
          description: User display name
          example: John Doe
        email:
          type: string
          format: email
          description: User email address
          example: john@example.com
        status:
          type: string
          enum:
            - ACTIVE
            - INACTIVE
            - SUSPENDED
          description: Current account status
          example: ACTIVE
```

#### Schema Rules

- Always include `description` for non-obvious fields
- Always include `example` values for documentation clarity
- Use `required` to mark mandatory fields explicitly
- Use `format` for standard types (email, date-time, uri, uuid)
- Use `readOnly` for fields that appear only in responses
- Use `writeOnly` for fields that appear only in requests (e.g., password)

### Error Response Documentation

#### Standard Error Schema

```yaml
components:
  schemas:
    ErrorResponse:
      type: object
      required:
        - error
      properties:
        error:
          type: object
          required:
            - code
            - message
          properties:
            code:
              type: string
              description: Machine-readable error code
              example: ENTITY_NOT_FOUND
            message:
              type: string
              description: Human-readable error message
              example: "User not found: 42"
            details:
              type: array
              items:
                $ref: "#/components/schemas/FieldError"
        meta:
          $ref: "#/components/schemas/ResponseMeta"
```

#### Error Documentation Rules

- Document all possible error responses for each endpoint
- Use `$ref` to reuse common error schemas
- Include realistic error examples
- Document error codes with their meanings in a central location

### Parameter Documentation

#### Path and Query Parameters

```yaml
parameters:
  - name: userId
    in: path
    required: true
    description: Unique user identifier
    schema:
      type: integer
      format: int64
      minimum: 1
    example: 42
  - name: status
    in: query
    required: false
    description: Filter by account status
    schema:
      type: string
      enum:
        - ACTIVE
        - INACTIVE
    example: ACTIVE
  - name: page
    in: query
    required: false
    description: Page number (1-based)
    schema:
      type: integer
      minimum: 1
      default: 1
    example: 1
```

#### Parameter Rules

- Always include `description` and `example`
- Specify `minimum`, `maximum`, `minLength`, `maxLength` constraints
- Use `default` for optional parameters with default values
- Define common parameters in `components/parameters` for reuse

### Authentication Documentation

```yaml
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: JWT access token

security:
  - BearerAuth: []
```

- Define security at the global level for default authentication
- Override per-endpoint for public endpoints (empty security array)
- Document token format and expiration in the description

### Versioning in Documentation

- Include API version in the spec `info.version`
- Document breaking changes between versions
- Maintain separate spec files per major version
- Mark deprecated endpoints with `deprecated: true` and migration notes

### Documentation Anti-Patterns

- Missing examples in schemas and parameters
- Undocumented error responses
- Inconsistent naming between spec and implementation
- Copy-pasted schemas instead of `$ref` references
- Missing authentication documentation
- No versioning strategy
- Stale documentation that does not match implementation

---

## 10. References

- [Semantic Versioning Specification](https://semver.org/) — 버전 번호 체계의 업계 표준
- [RFC 7231 (HTTP/1.1 Semantics and Content)](https://datatracker.ietf.org/doc/html/rfc7231) — HTTP 메서드 및 콘텐츠 협상
- [RFC 8594 (The Sunset HTTP Header Field)](https://datatracker.ietf.org/doc/html/rfc8594) — API 지원 종료 알림 표준
- [Microsoft REST API Guidelines - Versioning](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md) — 대규모 API 버저닝 사례
- [Google Cloud API Design Guide](https://cloud.google.com/apis/design) — API 호환성 및 버저닝 가이드
- [Pact Contract Testing](https://docs.pact.io/) — Consumer-Driven Contract Testing 프레임워크
