---
name: security
description: >-
  Framework-agnostic security rules including input validation, auth principles,
  CORS, API headers, rate limiting, and secret management.
  Use when implementing security-related code.
---
# Security Rules

## 1. Input Validation Principles

| Rule                                  | Purpose                                  |
| ------------------------------------- | ---------------------------------------- |
| Validate at API boundary              | Reject bad input early                   |
| Whitelist over blacklist              | Allow known-good, reject everything else |
| Validate type, length, range, format  | Prevent injection and overflow           |
| Sanitize output, not just input       | Prevent XSS in responses                 |
| Never trust client-side validation    | Always re-validate server-side           |

---

## 2. Authentication and Authorization Principles

- Apply least privilege — grant minimum permissions needed
- Use role-based access control (RBAC) at endpoint level
- Apply defense in depth — check authorization in service layer, not just URL
- Use method-level security for fine-grained control
- Log all authentication failures and authorization denials
- Never rely on URL-based security alone

---

## 3. CORS Principles

- Never use wildcard (`*`) origins in production
- Explicitly list allowed origins, methods, and headers
- Set `maxAge` to reduce preflight requests
- Separate CORS config per environment (dev may be more permissive)

---

## 4. API Security Headers

| Header                       | Value                                 | Purpose                   |
| ---------------------------- | ------------------------------------- | ------------------------- |
| `X-Content-Type-Options`     | `nosniff`                             | Prevent MIME sniffing     |
| `X-Frame-Options`            | `DENY`                                | Prevent clickjacking      |
| `Strict-Transport-Security`  | `max-age=31536000; includeSubDomains` | Force HTTPS               |
| `Cache-Control`              | `no-store`                            | Prevent sensitive caching |
| `X-XSS-Protection`           | `0`                                   | Disable (use CSP instead) |

---

## 5. Rate Limiting Guidelines

### Recommended Limits

| Endpoint Type      | Limit    | Window     |
| ------------------ | -------- | ---------- |
| Public API         | 100 req  | Per minute |
| Authenticated API  | 1000 req | Per minute |
| Login/Auth         | 10 req   | Per minute |
| File upload        | 10 req   | Per hour   |

### Response Headers

- `X-RateLimit-Limit`: Maximum requests allowed in window
- `X-RateLimit-Remaining`: Requests remaining in current window
- `X-RateLimit-Reset`: Timestamp when the window resets

---

## 6. Sensitive Data in Responses

### Never Expose

- Password hashes
- Internal IDs when external IDs exist
- Stack traces or internal error details
- Database column names in error messages
- Server version or framework information

### Response Filtering

- Use dedicated response DTOs — never return entities directly
- Exclude internal fields (password, internal flags, audit metadata)
- Map entities to response objects at the API boundary

---

## 7. Secret Management Principles

- Store secrets in environment variables or secret manager (Vault, AWS SSM, etc.)
- Never commit secrets to version control
- Rotate secrets periodically (at least every 90 days)
- Use different secrets per environment
- Revoke and rotate immediately if any secret is exposed
- Never provide default values for secrets in configuration files

---

## 8. Anti-Patterns

- Hardcoding secrets in source code or config files
- Returning entities directly from API endpoints
- Using wildcard CORS in production
- Missing rate limiting on authentication endpoints
- Logging sensitive data (passwords, tokens, PII)
- Trusting client-side validation without server-side checks
- Exposing detailed error internals in API responses
