# SRG-015 Query Optimization

## Definition

Query optimization is the process of improving database query performance by analyzing execution plans, creating appropriate indexes, and rewriting queries to reduce resource consumption and execution time.

## EXPLAIN ANALYZE

### PostgreSQL

```sql
-- Basic explain
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';

-- With execution statistics
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

-- Full analysis with buffers
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM users WHERE email = 'test@example.com';

-- JSON format for tools
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM users WHERE email = 'test@example.com';
```

### Understanding EXPLAIN Output

```sql
-- Example output
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;

/*
Seq Scan on orders  (cost=0.00..1234.00 rows=100 width=200) (actual time=0.015..12.500 rows=95 loops=1)
  Filter: (user_id = 123)
  Rows Removed by Filter: 9905
Planning Time: 0.100 ms
Execution Time: 12.600 ms
*/
```

**Key metrics:**
- `cost=0.00..1234.00` - estimated startup and total cost
- `rows=100` - estimated rows returned
- `actual time=0.015..12.500` - actual startup and total time (ms)
- `rows=95` - actual rows returned
- `loops=1` - number of loop iterations

### MySQL

```sql
-- Basic explain
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';

-- Extended explain
EXPLAIN EXTENDED SELECT * FROM users WHERE email = 'test@example.com';

-- JSON format
EXPLAIN FORMAT=JSON SELECT * FROM users WHERE email = 'test@example.com';

-- Analyze execution
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```

## Scan Types

### Sequential Scan (Seq Scan)

```sql
-- Bad: Full table scan
Seq Scan on users  (cost=0.00..10000.00 rows=100000 width=100)

-- When it happens:
-- 1. No suitable index
-- 2. Query returns large portion of table (>10-20%)
-- 3. Small table where seq scan is faster
```

### Index Scan

```sql
-- Good: Using index
Index Scan using users_email_idx on users  (cost=0.42..8.44 rows=1 width=100)
  Index Cond: (email = 'test@example.com'::text)

-- Reads index, then fetches rows from table
```

### Index Only Scan

```sql
-- Best: All data from index, no table access
Index Only Scan using users_email_name_idx on users  (cost=0.42..4.44 rows=1 width=50)
  Index Cond: (email = 'test@example.com'::text)

-- Requires: all selected columns in index + recent VACUUM
```

### Bitmap Scan

```sql
-- Good for multiple conditions or range queries
Bitmap Heap Scan on orders  (cost=50.00..500.00 rows=1000 width=100)
  Recheck Cond: (status = 'pending')
  ->  Bitmap Index Scan on orders_status_idx  (cost=0.00..49.75 rows=1000 width=0)
        Index Cond: (status = 'pending')

-- Two-phase: build bitmap, then fetch rows
```

## Index Types

### B-Tree Index (Default)

```sql
-- Best for: equality and range queries
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_date ON orders(created_at);

-- Supports: =, <, >, <=, >=, BETWEEN, IN, IS NULL
-- Also: LIKE 'prefix%' (but not '%suffix')
```

### Composite Index

```sql
-- Multi-column index
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Order matters! Index can be used for:
-- WHERE user_id = 123
-- WHERE user_id = 123 AND status = 'pending'
-- Cannot efficiently use for:
-- WHERE status = 'pending' (without user_id)
```

### Partial Index

```sql
-- Index subset of rows
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';

-- Smaller index, faster for specific queries
-- Only useful when filtering by the condition
```

### Expression Index

```sql
-- Index on expression result
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Query must use same expression
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';
```

### GIN Index (PostgreSQL)

```sql
-- For array, JSONB, full-text search
CREATE INDEX idx_posts_tags ON posts USING GIN(tags);
CREATE INDEX idx_data_jsonb ON data USING GIN(metadata jsonb_path_ops);

-- Full-text search
CREATE INDEX idx_articles_search ON articles USING GIN(to_tsvector('english', title || ' ' || body));
```

### Hash Index

```sql
-- Only for equality comparisons
CREATE INDEX idx_sessions_token ON sessions USING HASH(token);

-- Faster than B-tree for equality, but:
-- - No range queries
-- - Not WAL-logged before PostgreSQL 10
```

## Common Performance Problems

### N+1 Problem

```sql
-- Bad: N+1 queries
-- 1 query for users
SELECT * FROM users LIMIT 10;
-- N queries for orders (one per user)
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 2;
-- ... repeated 10 times

-- Good: JOIN or subquery
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
LIMIT 10;

-- Or batch query
SELECT * FROM orders WHERE user_id IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
```

### Missing Index

```sql
-- Identify missing indexes (PostgreSQL)
SELECT relname AS table_name,
       seq_scan,
       seq_tup_read,
       idx_scan,
       seq_tup_read / seq_scan AS avg_rows_per_scan
FROM pg_stat_user_tables
WHERE seq_scan > 100
  AND seq_tup_read / seq_scan > 1000
ORDER BY seq_tup_read DESC;
```

### Unused Index

```sql
-- Find unused indexes
SELECT schemaname, relname, indexrelname,
       idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS idx_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Implicit Type Conversion

```sql
-- Bad: Type mismatch prevents index use
-- user_id is INTEGER, but comparing with STRING
SELECT * FROM orders WHERE user_id = '123';

-- Good: Matching types
SELECT * FROM orders WHERE user_id = 123;
```

### Function on Indexed Column

```sql
-- Bad: Function prevents index use
SELECT * FROM users WHERE YEAR(created_at) = 2024;

-- Good: Range query uses index
SELECT * FROM users
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
```

### LIKE with Leading Wildcard

```sql
-- Bad: Cannot use index
SELECT * FROM users WHERE email LIKE '%@gmail.com';

-- Good: Can use index
SELECT * FROM users WHERE email LIKE 'john%';

-- Alternative: Use full-text search or trigram index
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_users_email_trgm ON users USING GIN(email gin_trgm_ops);
SELECT * FROM users WHERE email LIKE '%@gmail.com';  -- Now uses index
```

## Query Rewriting Techniques

### Use EXISTS Instead of IN

```sql
-- Slower: IN with subquery
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 1000);

-- Faster: EXISTS
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.total > 1000);
```

### Avoid SELECT *

```sql
-- Bad: Fetches all columns
SELECT * FROM users WHERE id = 123;

-- Good: Fetch only needed columns (enables index-only scan)
SELECT id, name, email FROM users WHERE id = 123;
```

### Limit Results Early

```sql
-- Bad: Sort all, then limit
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;

-- Better with index: Index provides order
CREATE INDEX idx_orders_created ON orders(created_at DESC);
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;  -- Index scan, no sort
```

### Use UNION ALL Instead of UNION

```sql
-- Slower: UNION removes duplicates (sorts)
SELECT email FROM users UNION SELECT email FROM contacts;

-- Faster: UNION ALL keeps duplicates (no sort)
SELECT email FROM users UNION ALL SELECT email FROM contacts;
```

### Optimize JOINs

```sql
-- Ensure indexes on join columns
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Put smaller table first (though optimizer usually handles this)
SELECT o.* FROM orders o
JOIN users u ON u.id = o.user_id
WHERE u.status = 'active';

-- For large joins, consider batch processing
```

## Analyzing Slow Queries

### PostgreSQL: pg_stat_statements

```sql
-- Top 10 by total time
SELECT query,
       calls,
       total_exec_time / 1000 AS total_sec,
       mean_exec_time / 1000 AS avg_sec,
       rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

### Slow Query Log

```sql
-- PostgreSQL (postgresql.conf)
log_min_duration_statement = 1000  -- Log queries > 1 second
log_statement = 'none'              -- Don't log all statements

-- MySQL (my.cnf)
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
```

## Index Maintenance

### Rebuild Bloated Indexes

```sql
-- Check index bloat (PostgreSQL)
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Rebuild index (blocking)
REINDEX INDEX idx_name;

-- Rebuild concurrently (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_name;
```

### Update Statistics

```sql
-- PostgreSQL
ANALYZE users;
ANALYZE;  -- All tables

-- MySQL
ANALYZE TABLE users;
```

## Best Practices

**Do:**
- Always use EXPLAIN ANALYZE for slow queries
- Create indexes for frequently filtered columns
- Use composite indexes for multi-column filters
- Keep statistics up to date (ANALYZE)
- Monitor query performance trends
- Use partial indexes for filtered queries
- Consider covering indexes for index-only scans
- Test index changes in staging first

**Don't:**
- Create too many indexes (slows writes)
- Use functions on indexed columns in WHERE
- Ignore implicit type conversions
- SELECT * when specific columns needed
- Use LIKE '%pattern%' without trigram index
- Forget to drop unused indexes
- Skip testing with production-like data volumes

## Performance Targets

```
# Response time targets
Fast queries: < 10ms
Normal queries: < 100ms
Complex queries: < 1 second
Reports/Analytics: < 10 seconds

# Index usage
Cache hit ratio: > 99%
Index hit ratio: > 95%
Seq scans on large tables: < 5% of queries
```

## Additional Resources

- [PostgreSQL EXPLAIN Documentation](https://www.postgresql.org/docs/current/using-explain.html)
- [Use The Index, Luke](https://use-the-index-luke.com/)
- [MySQL Query Optimization](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
- [pgMustard - EXPLAIN Analyzer](https://www.pgmustard.com/)
- [explain.depesz.com](https://explain.depesz.com/)
