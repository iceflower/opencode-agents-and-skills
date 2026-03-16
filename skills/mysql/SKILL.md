---
name: mysql
description: >-
  MySQL-specific rules including storage engines, data types, index design,
  query optimization, partitioning, replication, and configuration.
  Use when writing MySQL-specific SQL or tuning MySQL databases.
---

# MySQL Rules

## 1. Storage Engine Selection

### InnoDB (Default)

- Use InnoDB for all transactional workloads — it is the default and recommended engine
- Supports row-level locking, ACID transactions, and crash recovery
- Use `FOREIGN KEY` constraints only with InnoDB

### When to Consider Alternatives

| Engine  | Use Case                        | Trade-off                          |
| ------- | ------------------------------- | ---------------------------------- |
| InnoDB  | General purpose, transactional  | Slightly more disk usage           |
| MyISAM  | Read-heavy, full-text (legacy)  | No transactions, table-level lock  |
| MEMORY  | Temporary lookup tables         | Data lost on restart               |
| ARCHIVE | Append-only log storage         | No updates or deletes              |

- Avoid MyISAM for new projects — InnoDB supports full-text search since MySQL 5.6

---

## 2. Data Types

### Numeric Types

| Type                 | Size    | Range                              | Use Case                  |
| -------------------- | ------- | ---------------------------------- | ------------------------- |
| `TINYINT`            | 1 byte  | -128 to 127                        | Boolean, small enum       |
| `INT`                | 4 bytes | -2.1B to 2.1B                      | General integer           |
| `BIGINT`             | 8 bytes | -9.2E18 to 9.2E18                  | Large IDs, timestamps     |
| `DECIMAL(p,s)`       | Varies  | Exact precision                    | Money, financial data     |
| `FLOAT` / `DOUBLE`   | 4/8     | Approximate                        | Scientific data only      |

### String Types

| Type           | Max Size | Use Case                                   |
| -------------- | -------- | ------------------------------------------ |
| `VARCHAR(n)`   | 65,535   | Variable-length text (names, emails)       |
| `TEXT`         | 65,535   | Long text without index needs              |
| `MEDIUMTEXT`   | 16 MB    | Large documents                            |
| `ENUM`         | 65,535   | Fixed set of values                        |
| `JSON`         | 1 GB     | Structured data with query support         |

### Type Selection Rules

- Use `DECIMAL` for money — never `FLOAT` or `DOUBLE`
- Use `BIGINT` for primary keys in high-volume tables
- Use `VARCHAR` with appropriate length — do not default to `VARCHAR(255)`
- Use `DATETIME` over `TIMESTAMP` if timezone conversion is not needed
- Use `JSON` type for semi-structured data that needs MySQL JSON functions

---

## 3. Index Design

### Index Types

| Type        | Syntax                                | Use Case                           |
| ----------- | ------------------------------------- | ---------------------------------- |
| B-Tree      | `INDEX (col)`                         | General purpose (default)          |
| Unique      | `UNIQUE INDEX (col)`                  | Enforce uniqueness                 |
| Composite   | `INDEX (col1, col2)`                  | Multi-column WHERE/ORDER BY        |
| Prefix      | `INDEX (col(20))`                     | Long VARCHAR/TEXT columns          |
| Full-text   | `FULLTEXT INDEX (col)`                | Text search                        |
| Covering    | `INDEX (col1, col2, col3)`            | Avoid table lookup (index-only)    |

### Composite Index Rules

- Follow leftmost prefix rule — `INDEX (a, b, c)` supports queries on `(a)`, `(a, b)`, `(a, b, c)` but NOT `(b, c)`
- Place high-selectivity columns first (columns with more distinct values)
- Place equality columns before range columns in composite indexes
- ORDER BY columns must match index column order

### Index Anti-Patterns

- Indexing columns with low cardinality (boolean, status with 2-3 values) as standalone indexes
- Creating redundant indexes — `INDEX (a)` is redundant if `INDEX (a, b)` exists
- Over-indexing write-heavy tables — each index slows INSERT/UPDATE/DELETE
- Using functions on indexed columns in WHERE — `WHERE YEAR(created_at) = 2024` cannot use index

---

## 4. Query Optimization

### EXPLAIN Analysis

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
```

| EXPLAIN Field | What to Check                                        |
| ------------- | ---------------------------------------------------- |
| `type`        | `const` > `ref` > `range` > `index` > `ALL`          |
| `key`         | Which index is used (NULL = no index)                |
| `rows`        | Estimated rows scanned (lower is better)             |
| `Extra`       | Watch for `Using filesort`, `Using temporary`        |

### Query Rules

- Always check EXPLAIN for queries on large tables before deploying
- Avoid `SELECT *` — specify only needed columns
- Use `LIMIT` with `ORDER BY` for top-N queries
- Avoid `OR` on different columns — use `UNION ALL` instead
- Use `INSERT ... ON DUPLICATE KEY UPDATE` for upsert operations

---

## 5. Partitioning

### Partition Types

| Type    | Key                    | Use Case                          |
| ------- | ---------------------- | --------------------------------- |
| RANGE   | Date column            | Time-series data, log tables      |
| LIST    | Enum/status column     | Region or category-based data     |
| HASH    | Integer column         | Even distribution                 |
| KEY     | Any column (auto-hash) | General distribution              |

### Partitioning Rules

- Partition only when tables exceed millions of rows and queries filter by partition key
- Always include partition key in WHERE clause — otherwise MySQL scans all partitions
- Partition key must be part of every unique index (including primary key)
- Monitor partition sizes — unbalanced partitions negate performance benefits

---

## 6. Replication and High Availability

### Replication Types

| Type            | Consistency      | Latency        | Use Case                   |
| --------------- | ---------------- | -------------- | -------------------------- |
| Async           | Eventual         | Lowest         | Read scaling, backup       |
| Semi-sync       | At least one ACK | Low-medium     | Balanced durability        |
| Group (InnoDB)  | Strong           | Higher         | Multi-primary HA           |

### Read/Write Splitting Rules

- Route all writes to primary
- Route reads to replica, accounting for replication lag
- Never read-after-write from replica without lag awareness
- Use `gtid_mode=ON` for reliable failover

---

## 7. Configuration Essentials

### Key Parameters

| Parameter                          | Recommended                    | Purpose                      |
| ---------------------------------- | ------------------------------ | ---------------------------- |
| `innodb_buffer_pool_size`          | 70-80% of available RAM        | Cache data and indexes       |
| `innodb_log_file_size`             | 1-2 GB                         | Write-ahead log size         |
| `max_connections`                  | Based on workload              | Connection limit             |
| `slow_query_log`                   | ON                             | Identify slow queries        |
| `long_query_time`                  | 1 (second)                     | Slow query threshold         |
| `innodb_flush_log_at_trx_commit`   | 1 (durability) or 2 (perf)     | Fsync behavior per commit    |

---

## 8. Anti-Patterns

- Using `FLOAT`/`DOUBLE` for monetary values
- Creating indexes on every column "just in case"
- Using `SELECT *` in application queries
- Not checking EXPLAIN for complex queries
- Ignoring slow query log in production
- Using `UTF8` charset instead of `UTF8MB4` (UTF8 is 3-byte, does not support emoji)
- Running ALTER TABLE on large tables without `pt-online-schema-change` or `gh-ost`
- Not setting `innodb_buffer_pool_size` (defaults are too low for production)
