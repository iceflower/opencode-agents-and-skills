---
name: logging
description: Logging standards and sensitive data handling. Use when writing logging
  code or reviewing logs.
---

# Logging Rules

## 1. Log Level Usage

### Level Definitions

| Level  | Purpose                           | Examples                              |
| ------ | --------------------------------- | ------------------------------------- |
| ERROR  | Critical issues needing attention | DB connection failed, payment error   |
| WARN   | Potential problems, attention     | Deprecated API call, retry in progress|
| INFO   | Major business events             | User login, order created             |
| DEBUG  | Development/debugging details     | Variable values, method entry/exit    |
| TRACE  | Very detailed debugging info      | Full call stack, timing info          |

### Level Selection Criteria

```text
ERROR: Service disruption or data loss possible
WARN: Normal operation but monitoring needed
INFO: Business flow tracking
DEBUG: Root cause analysis when issues occur
TRACE: Performance analysis or deep debugging
```

---

## 2. Log Message Format

### Basic Structure

```text
[timestamp] [level] [traceId] [class:method] - message {context}
```

### Example

```text
2024-01-15T10:30:45.123Z INFO [abc123] [UserService:login] - User login successful {"userId": "user-001", "ip": "192.168.1.1"}
2024-01-15T10:31:00.456Z ERROR [abc123] [PaymentService:process] - Payment failed {"orderId": "order-123", "errorCode": "INSUFFICIENT_BALANCE"}
```

### Required Fields

| Field      | Description                       |
| ---------- | --------------------------------- |
| timestamp  | ISO 8601 format, UTC preferred    |
| level      | Log level                         |
| traceId    | Unique ID for request tracing     |
| location   | ClassName:methodName              |
| message    | Human-readable message            |
| context    | Additional context in JSON format |

---

## 3. Sensitive Data Handling

### Never Include in Logs

- Passwords
- API keys, secret keys
- Access tokens, refresh tokens
- Credit card numbers, CVV
- National ID, passport numbers
- Bank account numbers
- Biometric data
- Precise location data

### Masking Rules

| Data Type      | Masking Method                  | Example                  |
| -------------- | ------------------------------- | ------------------------ |
| Email          | First 2 chars + *** + @ + domain| `ab***@example.com`      |
| Phone          | Mask middle 4 digits            | `010-****-1234`          |
| Name           | First char + ***                | `J***`                   |
| Credit Card    | Show last 4 digits only         | `****-****-****-1234`    |
| API Key        | First 4 chars + ***             | `sk-****...`             |
| Address        | Show city/region only           | `Seoul ***`              |
| IP Address     | Mask last octet                 | `192.168.1.***`          |

### Masking Implementation Example

```typescript
// Email masking
function maskEmail(email: string): string {
  const [localPart, domain] = email.split('@');
  const masked = localPart.slice(0, 2) + '***';
  return `${masked}@${domain}`;
}

// Phone masking
function maskPhone(phone: string): string {
  return phone.replace(/(\d{3})-(\d{4})-(\d{4})/, '$1-****-$3');
}

// Credit card masking
function maskCardNumber(cardNumber: string): string {
  const lastFour = cardNumber.slice(-4);
  return `****-****-****-${lastFour}`;
}
```

---

## 4. Structured Logging

### JSON Log Format

```json
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "INFO",
  "traceId": "abc123",
  "service": "user-service",
  "class": "UserService",
  "method": "login",
  "message": "User login successful",
  "context": {
    "userId": "user-001",
    "loginMethod": "password",
    "durationMs": 150
  }
}
```

### Context Information Inclusion

| Info        | Include      | Note                      |
| ----------- | ------------ | ------------------------- |
| userId      | Yes          | No masking needed         |
| requestId   | Yes          | For tracing               |
| duration    | Yes          | Performance monitoring    |
| stack trace | Yes (ERROR)  | Error debugging           |
| password    | No           | Never include             |
| token       | No           | Never include             |

---

## 5. Logging Anti-Patterns

### Patterns to Avoid

```typescript
// Bad: Exposing sensitive info
logger.info(`User login: ${email}, password: ${password}`);

// Good: Exclude sensitive info
logger.info(`User login successful`, { userId: user.id });

// Bad: Error log without stack trace
logger.error(`Payment failed: ${error.message}`);

// Good: Include stack trace
logger.error(`Payment failed`, { error: error.stack, orderId });

// Bad: Excessive DEBUG logs
logger.debug(`Processing item 1`);
logger.debug(`Processing item 2`);
// ... thousands of logs

// Good: Log in meaningful batches
logger.debug(`Processing batch`, { itemCount: items.length });

// Bad: Wrong log level usage
logger.error(`User not found: ${userId}`);  // Business exception, not ERROR

// Good: Appropriate level
logger.info(`User not found`, { userId });  // Or WARN
```

---

## 6. Performance Considerations

- Disable DEBUG/TRACE in production by default
- Log in batches for large data processing
- Always mask sensitive info regardless of log level
- Use structured logs (JSON) for efficient searching

### Log Volume Guidelines

| Environment | INFO+ | DEBUG | TRACE |
| ----------- | ----- | ----- | ----- |
| Development | Yes   | Yes   | Yes   |
| Staging     | Yes   | Yes   | No    |
| Production  | Yes   | No    | No    |
