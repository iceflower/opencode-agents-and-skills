---
name: code-review
description: Code review checklist and comment guidelines. Use when reviewing pull
  requests or code changes.
---
# Code Review Rules

## 1. Review Checklist

### Correctness

- Does the code do what it claims to do?
- Are edge cases handled (null, empty, boundary values)?
- Are error paths handled properly?
- Is the logic correct for concurrent/async scenarios?

### Security

- Is user input validated and sanitized?
- Are sensitive data (passwords, tokens) excluded from logs and responses?
- Are authorization checks in place?
- Are SQL queries parameterized (no string concatenation)?

### Performance

- Are there N+1 query problems?
- Are large collections paginated?
- Are expensive operations cached where appropriate?
- Are database indexes considered for new query patterns?

### Readability

- Are names descriptive and consistent with existing conventions?
- Is the code self-explanatory without excessive comments?
- Is nesting depth reasonable (3 levels max)?
- Are magic numbers replaced with named constants?

### Testability

- Is the change covered by tests?
- Are tests testing behavior, not implementation?
- Are new edge cases included in tests?
- Can dependencies be easily replaced in tests?

---

## 2. Review Priorities

### Must Fix (block merge)

- Bugs or incorrect logic
- Security vulnerabilities
- Data loss risk
- Breaking API contract changes without versioning
- Missing error handling for critical paths

### Should Fix (strongly recommend)

- N+1 queries or obvious performance issues
- Missing input validation
- Duplicate code that should be extracted
- Missing tests for complex logic

### Nice to Have (optional, suggest)

- Minor naming improvements
- Code style preferences
- Additional documentation
- Alternative implementation approaches

---

## 3. Review Comment Style

### Good Comments

```text
// Specific, actionable, with context
"This query will cause N+1 when users have orders.
Consider using @EntityGraph or JOIN FETCH."

// Suggest, don't demand
"Consider using `sealed class` here — it would make
the when-expression exhaustive and catch missing cases at compile time."

// Ask questions to understand intent
"Is this intentional? If the token is expired, this returns null
instead of throwing — which means the caller needs to handle null."
```

### Bad Comments

```text
// Vague
"This doesn't look right."

// Nitpicking without value
"Add a blank line here."

// Prescriptive without rationale
"Use a different pattern."
```

---

## 4. Review Scope

### What to Review

- Business logic changes
- API contract changes (request/response, status codes)
- Database schema changes (migrations)
- Security-related code (auth, validation, data handling)
- Configuration changes (application.yml, build files)

### What to Skip

- Auto-generated code (unless the generator config changed)
- Dependency lock files (verify dependency changes only)
- IDE configuration files
- Formatting-only changes (should be handled by linter)

---

## 5. Self-Review Before Requesting Review

### Pre-PR Checklist

- [ ] Diff reviewed — no debug code, no commented-out code
- [ ] Tests pass locally
- [ ] No unintended file changes
- [ ] Commit messages follow convention
- [ ] PR description explains what and why
- [ ] Breaking changes documented
