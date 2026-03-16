---
name: database
description: >-
  Framework-agnostic database rules including migration conventions, query
  performance, transaction management, and anti-patterns.
  Use when writing SQL or designing database access.
---

# Database Rules

## 1. Migration File Conventions

### Naming Format

```text
V{version}__{description}.sql
```

| Element     | Rule                           | Example              |
| ----------- | ------------------------------ | -------------------- |
| Version     | Sequential number or timestamp | `V1`, `V20240115`    |
| Separator   | Double underscore `__`         | `V1__`               |
| Description | Snake_case, descriptive        | `create_users_table` |

### Migration Examples

```text
V1__create_users_table.sql
V2__add_email_index_to_users.sql
V3__create_orders_table.sql
V4__add_status_column_to_orders.sql
```

### Migration Rules

- Never modify a migration that has been applied to any environment
- Each migration must be idempotent where possible (use `IF NOT EXISTS`, `IF EXISTS`)
- Include both schema and essential seed data in migrations
- Test migrations against a copy of production data before applying

---

## 2. Query Performance

### Index Guidelines

- Add indexes on columns used in `WHERE`, `JOIN`, `ORDER BY`
- Use composite indexes for multi-column queries (column order matters)
- Avoid over-indexing — each index slows down writes
- Monitor slow query logs to identify missing indexes

### Query Best Practices

- Use pagination for list queries — never fetch unbounded result sets
- Avoid `SELECT *` — specify only needed columns for large tables
- Use `EXISTS` instead of `COUNT` for existence checks
- Avoid complex subqueries — prefer `JOIN` for readability and performance

---

## 3. Transaction Management Principles

### General Rules

| Rule                                      | Reason                              |
| ----------------------------------------- | ----------------------------------- |
| Keep transactions as short as possible    | Reduces lock contention             |
| No external API calls inside transactions | Prevents long-held locks on timeout |
| Use read-only transactions for reads      | Enables query optimizations         |
| Default to reusing existing transaction   | Avoids unnecessary overhead         |
| Use independent transactions sparingly    | Can cause deadlocks                 |

### Scope Example

```text
1. Read data (inside transaction or read-only)
2. Call external API (outside transaction)
3. Write result (separate transaction)
```

---

## 4. Anti-Patterns

- Unbounded queries without pagination
- `SELECT *` on large tables
- N+1 query patterns (loop of individual queries)
- Long-running transactions with external calls
- Missing indexes on frequently queried columns
- Modifying applied migration files
- No slow query monitoring
