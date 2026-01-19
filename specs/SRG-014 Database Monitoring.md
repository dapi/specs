# SRG-014 Database Monitoring & Metrics

## Definition

Database monitoring is the continuous process of tracking database health, performance, and resource utilization. It enables proactive identification of issues before they impact applications.

## Key Metrics Categories

### Connection Metrics

```
# PostgreSQL
db_connections_active (gauge)          # Active connections
db_connections_idle (gauge)            # Idle connections
db_connections_idle_in_transaction (gauge) # Idle in transaction
db_connections_max (gauge)             # max_connections setting
db_connections_used_percent (gauge)    # Usage percentage

# Thresholds
WARNING: connections_used > 70%
CRITICAL: connections_used > 85%
```

```sql
-- PostgreSQL: Current connections
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;

-- Connection age
SELECT pid, now() - backend_start AS connection_age,
       state, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY connection_age DESC;
```

### Query Performance Metrics

```
db_query_duration_seconds (histogram)  # Query execution time
db_queries_total (counter)             # Total queries executed
db_slow_queries_total (counter)        # Queries > threshold
db_rows_returned (counter)             # Rows returned
db_rows_affected (counter)             # Rows inserted/updated/deleted

# Thresholds
SLOW_QUERY_THRESHOLD: 1 second
WARNING: avg_query_time > 100ms
CRITICAL: avg_query_time > 500ms
```

### Transaction Metrics

```
db_transactions_committed (counter)    # Committed transactions
db_transactions_rolled_back (counter)  # Rolled back transactions
db_transaction_duration_seconds (histogram) # Transaction duration
db_deadlocks_total (counter)           # Deadlock count
db_lock_wait_seconds (histogram)       # Lock wait time

# Thresholds
WARNING: rollback_rate > 1%
CRITICAL: deadlocks > 0 per minute
```

### Storage Metrics

```
db_size_bytes (gauge)                  # Database size
db_table_size_bytes (gauge)            # Table size
db_index_size_bytes (gauge)            # Index size
db_bloat_bytes (gauge)                 # Table/index bloat
db_disk_usage_percent (gauge)          # Disk usage

# Thresholds
WARNING: disk_usage > 70%
CRITICAL: disk_usage > 85%
WARNING: bloat > 20%
```

### Cache Metrics

```
db_cache_hit_ratio (gauge)             # Buffer cache hit ratio
db_index_hit_ratio (gauge)             # Index hit ratio
db_shared_buffers_used (gauge)         # Shared buffers usage

# Thresholds
WARNING: cache_hit_ratio < 95%
CRITICAL: cache_hit_ratio < 90%
```

## PostgreSQL Specific Monitoring

### pg_stat_statements Extension

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Configuration (postgresql.conf)
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
pg_stat_statements.max = 10000

-- Top 10 slowest queries
SELECT query,
       calls,
       total_exec_time / 1000 AS total_seconds,
       mean_exec_time / 1000 AS avg_seconds,
       rows,
       100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Most called queries
SELECT query, calls, rows, mean_exec_time
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- Queries consuming most time
SELECT query,
       total_exec_time / 1000 AS total_seconds,
       calls,
       mean_exec_time / 1000 AS avg_seconds
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

### pg_stat_user_tables

```sql
-- Table statistics
SELECT relname AS table_name,
       seq_scan,
       seq_tup_read,
       idx_scan,
       idx_tup_fetch,
       n_tup_ins AS inserts,
       n_tup_upd AS updates,
       n_tup_del AS deletes,
       n_live_tup AS live_rows,
       n_dead_tup AS dead_rows,
       last_vacuum,
       last_autovacuum,
       last_analyze,
       last_autoanalyze
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;

-- Tables needing index (high seq_scan, low idx_scan)
SELECT relname,
       seq_scan,
       idx_scan,
       seq_scan - idx_scan AS seq_vs_idx
FROM pg_stat_user_tables
WHERE seq_scan > 1000
ORDER BY seq_vs_idx DESC;
```

### pg_stat_user_indexes

```sql
-- Unused indexes
SELECT schemaname, relname, indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS idx_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Index usage statistics
SELECT relname AS table_name,
       indexrelname AS index_name,
       idx_scan,
       idx_tup_read,
       idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

### Lock Monitoring

```sql
-- Current locks
SELECT blocked_locks.pid AS blocked_pid,
       blocked_activity.usename AS blocked_user,
       blocking_locks.pid AS blocking_pid,
       blocking_activity.usename AS blocking_user,
       blocked_activity.query AS blocked_query,
       blocking_activity.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- Long running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes'
  AND state != 'idle';
```

## MySQL Specific Monitoring

### Performance Schema

```sql
-- Enable Performance Schema (my.cnf)
performance_schema = ON

-- Top queries by total time
SELECT DIGEST_TEXT,
       COUNT_STAR AS calls,
       SUM_TIMER_WAIT / 1000000000000 AS total_seconds,
       AVG_TIMER_WAIT / 1000000000000 AS avg_seconds,
       SUM_ROWS_EXAMINED,
       SUM_ROWS_SENT
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- Table I/O statistics
SELECT OBJECT_SCHEMA, OBJECT_NAME,
       COUNT_READ, COUNT_WRITE,
       SUM_TIMER_READ / 1000000000000 AS read_seconds,
       SUM_TIMER_WRITE / 1000000000000 AS write_seconds
FROM performance_schema.table_io_waits_summary_by_table
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

### InnoDB Metrics

```sql
-- InnoDB buffer pool hit ratio
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';

-- Calculate hit ratio
SELECT
    (1 - (
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    )) * 100 AS buffer_pool_hit_ratio;

-- InnoDB row operations
SHOW GLOBAL STATUS LIKE 'Innodb_rows%';
```

## Prometheus Exporters

### postgres_exporter

```yaml
# docker-compose.yml
services:
  postgres_exporter:
    image: prometheuscommunity/postgres-exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://user:pass@db:5432/mydb?sslmode=disable"
    ports:
      - "9187:9187"
```

### mysqld_exporter

```yaml
# docker-compose.yml
services:
  mysqld_exporter:
    image: prom/mysqld-exporter
    environment:
      DATA_SOURCE_NAME: "user:pass@(db:3306)/"
    ports:
      - "9104:9104"
```

## Grafana Dashboards

### Essential Panels

```
1. Connection Pool Status
   - Active/Idle/Max connections
   - Connection usage percentage

2. Query Performance
   - Queries per second
   - Average query latency
   - Slow queries count

3. Transaction Metrics
   - Transactions per second
   - Commit/Rollback ratio
   - Deadlock count

4. Storage
   - Database size
   - Table sizes (top 10)
   - Disk usage

5. Cache Performance
   - Buffer cache hit ratio
   - Index hit ratio

6. Replication (if applicable)
   - Replication lag
   - Replica status
```

### Dashboard JSON Example

```json
{
  "dashboard": {
    "title": "PostgreSQL Overview",
    "panels": [
      {
        "title": "Connections",
        "type": "gauge",
        "targets": [
          {
            "expr": "pg_stat_activity_count / pg_settings_max_connections * 100"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 70},
                {"color": "red", "value": 85}
              ]
            }
          }
        }
      }
    ]
  }
}
```

## Alerting Rules

```yaml
groups:
  - name: database
    rules:
      # Connection alerts
      - alert: DatabaseConnectionsHigh
        expr: pg_stat_activity_count / pg_settings_max_connections > 0.7
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Database connections above 70%"

      - alert: DatabaseConnectionsCritical
        expr: pg_stat_activity_count / pg_settings_max_connections > 0.85
        for: 2m
        labels:
          severity: critical

      # Performance alerts
      - alert: SlowQueries
        expr: rate(pg_stat_statements_seconds_total[5m]) > 1
        for: 10m
        labels:
          severity: warning

      - alert: DatabaseDeadlock
        expr: increase(pg_stat_database_deadlocks[5m]) > 0
        labels:
          severity: critical

      # Storage alerts
      - alert: DatabaseDiskUsageHigh
        expr: pg_database_size_bytes / node_filesystem_size_bytes > 0.7
        for: 10m
        labels:
          severity: warning

      # Cache alerts
      - alert: LowCacheHitRatio
        expr: pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read) < 0.95
        for: 15m
        labels:
          severity: warning

      # Replication alerts
      - alert: ReplicationLagHigh
        expr: pg_replication_lag > 30
        for: 5m
        labels:
          severity: warning
```

## Monitoring Tools

| Tool | Description | Use Case |
|------|-------------|----------|
| pg_stat_statements | Query statistics | Query optimization |
| pgBadger | Log analyzer | Historical analysis |
| pg_activity | Live monitoring | Real-time debugging |
| Percona PMM | Full monitoring | Enterprise monitoring |
| DataDog | Cloud monitoring | SaaS solution |
| New Relic | APM + DB | Application correlation |

## Best Practices

**Do:**
- Enable pg_stat_statements in all environments
- Set up alerting for critical metrics
- Create dashboards for key metrics
- Retain metrics history for capacity planning
- Correlate DB metrics with application metrics
- Monitor both aggregate and per-query metrics
- Track slow query trends over time

**Don't:**
- Ignore cache hit ratio degradation
- Skip connection pool monitoring
- Forget to monitor disk space
- Over-alert on temporary spikes
- Ignore index usage statistics
- Skip lock monitoring

## Environment Variables

```bash
# Monitoring configuration
DB_SLOW_QUERY_THRESHOLD_MS=1000
DB_MONITORING_INTERVAL_SECONDS=15
DB_STATS_RETENTION_DAYS=30

# Alerting thresholds
DB_ALERT_CONNECTIONS_WARNING=70
DB_ALERT_CONNECTIONS_CRITICAL=85
DB_ALERT_CACHE_HIT_WARNING=95
DB_ALERT_DISK_USAGE_WARNING=70
DB_ALERT_REPLICATION_LAG_WARNING=30
```

## Additional Resources

- [PostgreSQL Statistics Collector](https://www.postgresql.org/docs/current/monitoring-stats.html)
- [MySQL Performance Schema](https://dev.mysql.com/doc/refman/8.0/en/performance-schema.html)
- [postgres_exporter](https://github.com/prometheus-community/postgres_exporter)
- [pgBadger](https://github.com/darold/pgbadger)
- [Percona Monitoring and Management](https://www.percona.com/software/database-tools/percona-monitoring-and-management)
