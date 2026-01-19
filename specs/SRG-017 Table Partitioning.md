# SRG-017 Table Partitioning

## Definition

Table partitioning divides a large table into smaller, more manageable pieces called partitions. Each partition stores a subset of the data based on partition key values. Unlike sharding, all partitions reside in the same database instance.

## When to Partition

**Use partitioning when:**
- Table has millions/billions of rows
- Queries filter by partition key (date, region)
- Need to efficiently archive or delete old data
- Want to improve query performance on large tables
- Need parallel query execution

**Avoid partitioning when:**
- Table is small (< 1M rows)
- Queries don't filter by partition key
- Complexity outweighs benefits

## Partitioning Strategies

### Range Partitioning

```sql
-- PostgreSQL: Partition by date range
CREATE TABLE events (
    id bigserial,
    event_type varchar(50),
    payload jsonb,
    created_at timestamptz NOT NULL
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
CREATE TABLE events_2024_03 PARTITION OF events
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- Default partition for unexpected values
CREATE TABLE events_default PARTITION OF events DEFAULT;
```

### List Partitioning

```sql
-- PostgreSQL: Partition by region
CREATE TABLE orders (
    id bigserial,
    customer_id int,
    region varchar(20) NOT NULL,
    total decimal(10,2),
    created_at timestamptz
) PARTITION BY LIST (region);

CREATE TABLE orders_us PARTITION OF orders FOR VALUES IN ('US', 'CA');
CREATE TABLE orders_eu PARTITION OF orders FOR VALUES IN ('UK', 'DE', 'FR');
CREATE TABLE orders_asia PARTITION OF orders FOR VALUES IN ('JP', 'CN', 'KR');
CREATE TABLE orders_other PARTITION OF orders DEFAULT;
```

### Hash Partitioning

```sql
-- PostgreSQL: Partition by hash for even distribution
CREATE TABLE sessions (
    id uuid PRIMARY KEY,
    user_id int NOT NULL,
    data jsonb,
    expires_at timestamptz
) PARTITION BY HASH (user_id);

CREATE TABLE sessions_0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE sessions_2 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE sessions_3 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

### Composite Partitioning

```sql
-- PostgreSQL: Range + List (sub-partitioning)
CREATE TABLE logs (
    id bigserial,
    log_level varchar(10) NOT NULL,
    message text,
    created_at timestamptz NOT NULL
) PARTITION BY RANGE (created_at);

-- Monthly partition
CREATE TABLE logs_2024_01 PARTITION OF logs
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01')
    PARTITION BY LIST (log_level);

-- Sub-partitions by log level
CREATE TABLE logs_2024_01_error PARTITION OF logs_2024_01
    FOR VALUES IN ('ERROR', 'FATAL');
CREATE TABLE logs_2024_01_info PARTITION OF logs_2024_01
    FOR VALUES IN ('INFO', 'DEBUG', 'WARN');
```

## MySQL Partitioning

```sql
-- Range partitioning by date
CREATE TABLE events (
    id BIGINT AUTO_INCREMENT,
    event_type VARCHAR(50),
    payload JSON,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (YEAR(created_at) * 100 + MONTH(created_at)) (
    PARTITION p202401 VALUES LESS THAN (202402),
    PARTITION p202402 VALUES LESS THAN (202403),
    PARTITION p202403 VALUES LESS THAN (202404),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);

-- List partitioning
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT,
    region VARCHAR(20) NOT NULL,
    total DECIMAL(10,2),
    PRIMARY KEY (id, region)
) PARTITION BY LIST COLUMNS (region) (
    PARTITION p_us VALUES IN ('US', 'CA'),
    PARTITION p_eu VALUES IN ('UK', 'DE', 'FR'),
    PARTITION p_asia VALUES IN ('JP', 'CN', 'KR')
);
```

## Partition Pruning

```sql
-- Query that benefits from partition pruning
SELECT * FROM events
WHERE created_at >= '2024-02-01' AND created_at < '2024-03-01';

-- EXPLAIN shows only events_2024_02 is scanned
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM events WHERE created_at = '2024-02-15';

/*
Append
  ->  Seq Scan on events_2024_02
        Filter: (created_at = '2024-02-15')
*/
```

## Automated Partition Management

### PostgreSQL: pg_partman Extension

```sql
-- Install pg_partman
CREATE EXTENSION pg_partman;

-- Create partitioned table
CREATE TABLE metrics (
    id bigserial,
    metric_name varchar(100),
    value numeric,
    recorded_at timestamptz NOT NULL
) PARTITION BY RANGE (recorded_at);

-- Configure automatic partition management
SELECT partman.create_parent(
    p_parent_table := 'public.metrics',
    p_control := 'recorded_at',
    p_type := 'native',
    p_interval := 'daily',
    p_premake := 7  -- Create 7 days ahead
);

-- Run maintenance (via cron or pg_cron)
SELECT partman.run_maintenance();
```

### Automatic Partition Creation Script

```sql
-- PostgreSQL function for automatic monthly partitions
CREATE OR REPLACE FUNCTION create_monthly_partition()
RETURNS void AS $$
DECLARE
    partition_date date;
    partition_name text;
    start_date date;
    end_date date;
BEGIN
    -- Create partitions for next 3 months
    FOR i IN 0..2 LOOP
        partition_date := date_trunc('month', CURRENT_DATE + (i || ' month')::interval);
        partition_name := 'events_' || to_char(partition_date, 'YYYY_MM');
        start_date := partition_date;
        end_date := partition_date + interval '1 month';

        -- Check if partition exists
        IF NOT EXISTS (
            SELECT 1 FROM pg_class WHERE relname = partition_name
        ) THEN
            EXECUTE format(
                'CREATE TABLE %I PARTITION OF events FOR VALUES FROM (%L) TO (%L)',
                partition_name, start_date, end_date
            );
            RAISE NOTICE 'Created partition: %', partition_name;
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Schedule with pg_cron
SELECT cron.schedule('create_partitions', '0 0 1 * *', 'SELECT create_monthly_partition()');
```

## Partition Maintenance

### Dropping Old Partitions

```sql
-- Drop partition (fast, no locks on other partitions)
DROP TABLE events_2023_01;

-- Detach first if needed to query before drop
ALTER TABLE events DETACH PARTITION events_2023_01;
-- ... backup or archive ...
DROP TABLE events_2023_01;
```

### Archiving Partitions

```sql
-- Detach partition for archiving
ALTER TABLE events DETACH PARTITION events_2023_12;

-- Move to archive tablespace
ALTER TABLE events_2023_12 SET TABLESPACE archive_tablespace;

-- Or export to file
COPY events_2023_12 TO '/archive/events_2023_12.csv' WITH CSV HEADER;

-- Then drop
DROP TABLE events_2023_12;
```

### Reattaching Partitions

```sql
-- Reattach partition (with constraint validation)
ALTER TABLE events ATTACH PARTITION events_2023_12
    FOR VALUES FROM ('2023-12-01') TO ('2024-01-01');
```

## Indexing Partitioned Tables

```sql
-- Index on partitioned table (creates index on each partition)
CREATE INDEX idx_events_type ON events (event_type);

-- Unique constraints must include partition key
CREATE UNIQUE INDEX idx_events_id ON events (id, created_at);

-- Local indexes (per partition)
CREATE INDEX idx_events_2024_01_payload ON events_2024_01 USING GIN (payload);
```

## Query Optimization

### Queries that Benefit from Partitioning

```sql
-- Good: Filters by partition key
SELECT * FROM events WHERE created_at >= '2024-02-01' AND created_at < '2024-03-01';

-- Good: Aggregates by partition key range
SELECT date_trunc('day', created_at) AS day, count(*)
FROM events
WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01'
GROUP BY 1;

-- Good: Deleting old data (via partition drop)
ALTER TABLE events DETACH PARTITION events_2023_01;
DROP TABLE events_2023_01;
```

### Queries that Don't Benefit

```sql
-- Bad: No partition key in WHERE (scans all partitions)
SELECT * FROM events WHERE event_type = 'login';

-- Fix: Add partition key to query
SELECT * FROM events
WHERE event_type = 'login'
  AND created_at >= '2024-01-01' AND created_at < '2024-02-01';
```

## Monitoring

```sql
-- List partitions and sizes
SELECT
    parent.relname AS parent_table,
    child.relname AS partition_name,
    pg_size_pretty(pg_relation_size(child.oid)) AS size,
    pg_stat_user_tables.n_live_tup AS row_count
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
LEFT JOIN pg_stat_user_tables ON child.relname = pg_stat_user_tables.relname
WHERE parent.relname = 'events'
ORDER BY child.relname;

-- Check partition pruning in query plan
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM events WHERE created_at = '2024-02-15';
```

## Best Practices

**Do:**
- Choose partition key based on query patterns
- Create partitions in advance (premake)
- Include partition key in all indexes
- Use partition pruning by filtering on partition key
- Automate partition creation and cleanup
- Monitor partition sizes for balance
- Test partition operations in staging

**Don't:**
- Partition small tables
- Use too many partitions (>1000 impacts planning)
- Forget to create default partition
- Query without partition key filter
- Create partitions manually without automation
- Ignore partition maintenance

## Environment Variables

```bash
# Partition configuration
DB_PARTITION_STRATEGY=range
DB_PARTITION_KEY=created_at
DB_PARTITION_INTERVAL=monthly
DB_PARTITION_PREMAKE=3
DB_PARTITION_RETENTION_MONTHS=12

# Maintenance
DB_PARTITION_MAINTENANCE_ENABLED=true
DB_PARTITION_MAINTENANCE_SCHEDULE="0 2 * * *"
```

## Additional Resources

- [PostgreSQL Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [pg_partman Extension](https://github.com/pgpartman/pg_partman)
- [MySQL Partitioning](https://dev.mysql.com/doc/refman/8.0/en/partitioning.html)
- [Partition Pruning](https://www.postgresql.org/docs/current/ddl-partitioning.html#DDL-PARTITION-PRUNING)
