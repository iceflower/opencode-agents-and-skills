---
name: postgresql
description: >-
  PostgreSQL-specific rules including data types (JSONB, UUID, arrays),
  index types (GIN, GiST, BRIN), CTEs, partitioning, and extensions.
  Use when writing PostgreSQL-specific SQL or tuning PostgreSQL databases.
---

# PostgreSQL Rules

## 1. Data Types

### PostgreSQL-Specific Types

| Type           | Use Case                                 | Advantage over Generic                   |
| -------------- | ---------------------------------------- | ---------------------------------------- |
| `UUID`         | Primary keys, distributed IDs            | Native UUID generation/indexing          |
| `JSONB`        | Semi-structured data                     | Binary storage, indexable                |
| `ARRAY`        | Multi-value columns                      | No join table needed                     |
| `INET`/`CIDR`  | IP addresses, network ranges             | Built-in network functions               |
| `TSTZRANGE`    | Time ranges (booking, scheduling)        | Overlap/containment operators            |
| `INTERVAL`     | Duration values                          | Date arithmetic support                  |
| `TEXT`         | Variable-length strings (no limit)       | No performance difference vs VARCHAR     |
| `NUMERIC(p,s)` | Exact precision (money, financial)       | Arbitrary precision                      |

### Type Selection Rules

- Use `BIGINT` or `UUID` for primary keys — avoid `SERIAL` (prefer `GENERATED ALWAYS AS IDENTITY`)
- Use `JSONB` over `JSON` — JSONB is binary, indexable, and faster for queries
- Use `TEXT` instead of `VARCHAR(n)` unless a strict length constraint is needed
- Use `TIMESTAMPTZ` (with timezone) over `TIMESTAMP` — always store timezone-aware
- Use `NUMERIC` for money — never `REAL` or `DOUBLE PRECISION`
- Use `BOOLEAN` natively — PostgreSQL has a true boolean type

---

## 2. Index Design

### Index Types

| Type    | Syntax                              | Use Case                            |
| ------- | ----------------------------------- | ----------------------------------- |
| B-tree  | `CREATE INDEX ... ON (col)`         | General purpose (default)           |
| Hash    | `CREATE INDEX ... USING hash (col)` | Equality-only lookups               |
| GIN     | `CREATE INDEX ... USING gin (col)`  | JSONB, arrays, full-text search     |
| GiST    | `CREATE INDEX ... USING gist (col)` | Geometric, range, full-text search  |
| BRIN    | `CREATE INDEX ... USING brin (col)` | Large naturally-ordered tables      |

### Advanced Index Features

```sql
-- Partial index: only index active users
CREATE INDEX idx_users_active_email ON users (email) WHERE status = 'ACTIVE';

-- Expression index: case-insensitive search
CREATE INDEX idx_users_lower_email ON users (lower(email));

-- Covering index: include extra columns to avoid table lookup
CREATE INDEX idx_orders_user ON orders (user_id) INCLUDE (status, created_at);

-- GIN index for JSONB
CREATE INDEX idx_metadata ON events USING gin (metadata jsonb_path_ops);
```

### Index Rules

- Use partial indexes to reduce index size when queries filter by a constant condition
- Use expression indexes for queries with functions (e.g., `lower()`, `date_trunc()`)
- Use GIN indexes for JSONB columns and array containment queries
- Use BRIN indexes for append-only tables with natural ordering (e.g., time-series)
- Use `CONCURRENTLY` for production index creation to avoid table locks

---

## 3. JSONB Usage

### Query Patterns

```sql
-- Access nested value
SELECT metadata->>'name' FROM events WHERE metadata->>'type' = 'click';

-- Containment operator (uses GIN index)
SELECT * FROM events WHERE metadata @> '{"type": "click"}';

-- Path exists
SELECT * FROM events WHERE metadata ? 'email';

-- Update nested value
UPDATE events SET metadata = jsonb_set(metadata, '{status}', '"processed"');
```

### JSONB Rules

- Use `@>` containment operator instead of `->>` for indexed queries
- Create GIN index with `jsonb_path_ops` for containment queries
- Do not store relational data in JSONB — use proper columns for frequently queried fields
- Use JSONB for truly semi-structured data (metadata, settings, external API responses)

---

## 4. Common Table Expressions (CTE)

### CTE Patterns

```sql
-- Readable multi-step queries
WITH active_users AS (
    SELECT id, name FROM users WHERE status = 'ACTIVE'
),
recent_orders AS (
    SELECT user_id, COUNT(*) AS order_count
    FROM orders
    WHERE created_at > NOW() - INTERVAL '30 days'
    GROUP BY user_id
)
SELECT u.name, COALESCE(o.order_count, 0) AS orders
FROM active_users u
LEFT JOIN recent_orders o ON u.id = o.user_id;

-- Recursive CTE for hierarchical data
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 1 AS depth
    FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree;
```

### CTE Rules

- CTEs in PostgreSQL 12+ are automatically inlined when not materialized — no performance penalty
- Use `MATERIALIZED` hint when CTE result is reused multiple times to avoid re-evaluation
- Use recursive CTEs for tree/graph traversal instead of application-level loops
- Avoid deeply nested CTEs — if readability decreases, consider views or functions

---

## 5. Partitioning

### Declarative Partitioning

```sql
-- Range partitioning by date
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    created_at TIMESTAMPTZ NOT NULL,
    amount NUMERIC(10,2)
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
```

### Partition Types

| Type   | Key                 | Use Case                            |
| ------ | ------------------- | ----------------------------------- |
| RANGE  | Date, numeric range | Time-series, audit logs             |
| LIST   | Enum, region        | Category or tenant-based data       |
| HASH   | Any column          | Even distribution across partitions |

### Partitioning Rules

- Use declarative partitioning (PostgreSQL 10+) — avoid inheritance-based partitioning
- Always include partition key in queries to enable partition pruning
- Create indexes on each partition individually (they are not inherited automatically before PG 11)
- Automate partition creation for time-based partitions (e.g., `pg_partman`)

---

## 6. Performance Features

### VACUUM and Autovacuum

- PostgreSQL uses MVCC — dead tuples accumulate and need vacuuming
- Keep autovacuum enabled — never disable it in production
- Tune `autovacuum_vacuum_scale_factor` for high-write tables (default 20% may be too high)
- Run `VACUUM ANALYZE` after large bulk data operations

### Connection Pooling

- Use PgBouncer or other external connection pooler (e.g., Pgpool-II) in production
- PostgreSQL forks a process per connection — too many connections waste memory
- Recommended: pool size = `(core_count * 2) + disk_spindles`

### EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = 123 AND status = 'ACTIVE';
```

| Key Metric            | What to Check                                  |
| --------------------- | ---------------------------------------------- |
| `Seq Scan`            | Full table scan — may need an index            |
| `Index Scan`          | Good — using index                             |
| `Bitmap Heap Scan`    | Acceptable for medium selectivity              |
| `Buffers: shared hit` | Cache hits — higher is better                  |
| `actual time`         | Real execution time per node                   |

---

## 7. Extensions

### Commonly Used Extensions

| Extension              | Purpose                              | Enable Command                          |
| ---------------------- | ------------------------------------ | --------------------------------------- |
| `uuid-ossp`            | UUID generation functions            | `CREATE EXTENSION "uuid-ossp"`          |
| `pgcrypto`             | Cryptographic functions              | `CREATE EXTENSION pgcrypto`             |
| `pg_trgm`              | Trigram-based text similarity search | `CREATE EXTENSION pg_trgm`              |
| `pg_stat_statements`   | Query performance statistics         | `CREATE EXTENSION pg_stat_statements`   |
| `postgis`              | Geospatial data and queries          | `CREATE EXTENSION postgis`              |
| `citext`               | Case-insensitive text type           | `CREATE EXTENSION citext`               |

- Always enable `pg_stat_statements` in production for query performance monitoring
- Use `pg_trgm` with GIN index for LIKE/ILIKE pattern matching performance

---

## 8. Anti-Patterns

- Using `SERIAL` instead of `GENERATED ALWAYS AS IDENTITY` (legacy syntax)
- Using `JSON` instead of `JSONB` (no indexing, no binary storage)
- Using `TIMESTAMP` without timezone (`TIMESTAMPTZ` is almost always correct)
- Using `VARCHAR(255)` by default — use `TEXT` or appropriate length
- Disabling autovacuum (causes table bloat and performance degradation)
- Not using `CONCURRENTLY` for production index creation (locks writes)
- Opening too many direct connections without a connection pooler
- Not monitoring with `pg_stat_statements`
