# SRO-008 Database Maintenance

## Definition

Database maintenance encompasses routine operations that keep databases healthy, performant, and reliable. This includes VACUUM operations, statistics updates, index maintenance, bloat management, and scheduled cleanup tasks.

## PostgreSQL Maintenance Operations

### VACUUM

```sql
-- Standard VACUUM (reclaims space, updates visibility map)
VACUUM tablename;

-- VACUUM with statistics update
VACUUM ANALYZE tablename;

-- Full VACUUM (rewrites entire table, requires exclusive lock)
VACUUM FULL tablename;

-- VACUUM specific columns for statistics
VACUUM ANALYZE tablename (column1, column2);

-- Check VACUUM progress (PostgreSQL 9.6+)
SELECT * FROM pg_stat_progress_vacuum;
```

### ANALYZE

```sql
-- Update statistics for query planner
ANALYZE tablename;

-- Analyze specific columns
ANALYZE tablename (column1, column2);

-- Analyze entire database
ANALYZE;

-- Check when last analyzed
SELECT
    schemaname,
    relname,
    last_analyze,
    last_autoanalyze,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
ORDER BY last_analyze NULLS FIRST;
```

### REINDEX

```sql
-- Reindex single index
REINDEX INDEX idx_users_email;

-- Reindex all indexes on a table
REINDEX TABLE users;

-- Reindex entire database
REINDEX DATABASE mydb;

-- Concurrent reindex (PostgreSQL 12+, no locks)
REINDEX INDEX CONCURRENTLY idx_users_email;
REINDEX TABLE CONCURRENTLY users;

-- Check index bloat before reindex
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS index_scans
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;
```

## Autovacuum Configuration

### postgresql.conf Settings

```ini
# Enable autovacuum
autovacuum = on

# Worker processes
autovacuum_max_workers = 3

# How often to run (seconds)
autovacuum_naptime = 60

# Thresholds for VACUUM
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.2

# Thresholds for ANALYZE
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.1

# Cost-based vacuum delay (throttling)
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 200

# Freeze age settings
vacuum_freeze_min_age = 50000000
vacuum_freeze_table_age = 150000000
autovacuum_freeze_max_age = 200000000
```

### Per-Table Settings

```sql
-- High-write table: more aggressive autovacuum
ALTER TABLE events SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_vacuum_threshold = 1000,
    autovacuum_analyze_scale_factor = 0.02,
    autovacuum_analyze_threshold = 500
);

-- Large table: increase freeze threshold
ALTER TABLE audit_logs SET (
    autovacuum_freeze_max_age = 500000000,
    autovacuum_freeze_table_age = 400000000
);

-- Disable autovacuum for specific table (use with caution)
ALTER TABLE temp_data SET (autovacuum_enabled = false);
```

## Bloat Management

### Detecting Table Bloat

```sql
-- Estimate table bloat
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS table_size,
    n_dead_tup,
    n_live_tup,
    CASE
        WHEN n_live_tup > 0
        THEN round(100.0 * n_dead_tup / (n_live_tup + n_dead_tup), 2)
        ELSE 0
    END AS dead_tuple_percent
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- More accurate bloat estimation using pgstattuple
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT
    table_len,
    tuple_count,
    tuple_len,
    dead_tuple_count,
    dead_tuple_len,
    free_space,
    free_percent
FROM pgstattuple('tablename');
```

### Detecting Index Bloat

```sql
-- Index bloat estimation
SELECT
    c.relname AS index_name,
    pg_size_pretty(pg_relation_size(c.oid)) AS index_size,
    s.idx_scan AS scans,
    s.idx_tup_read AS tuples_read,
    s.idx_tup_fetch AS tuples_fetched
FROM pg_class c
JOIN pg_stat_user_indexes s ON c.relname = s.indexrelname
WHERE c.relkind = 'i'
ORDER BY pg_relation_size(c.oid) DESC;

-- Using pgstattuple for index bloat
SELECT * FROM pgstatindex('idx_users_email');
```

### pg_repack Extension

```sql
-- Install pg_repack
CREATE EXTENSION pg_repack;

-- Repack table (no locks, online operation)
-- Run from command line:
-- pg_repack -d mydb -t tablename

-- Repack with specific index
-- pg_repack -d mydb -t tablename -i idx_name

-- Repack entire database
-- pg_repack -d mydb
```

## Maintenance Windows

### Scheduled Maintenance Script

```bash
#!/bin/bash
# maintenance.sh - Database maintenance script

DB_NAME="mydb"
LOG_FILE="/var/log/db_maintenance.log"
LOCK_FILE="/var/run/db_maintenance.lock"

# Prevent concurrent runs
exec 200>$LOCK_FILE
flock -n 200 || { echo "Maintenance already running"; exit 1; }

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> $LOG_FILE
}

log "Starting maintenance for $DB_NAME"

# 1. VACUUM ANALYZE high-churn tables
log "Running VACUUM ANALYZE on high-churn tables"
psql -d $DB_NAME -c "VACUUM ANALYZE events, sessions, audit_logs;"

# 2. Update statistics on all tables
log "Running ANALYZE on database"
psql -d $DB_NAME -c "ANALYZE;"

# 3. Reindex bloated indexes (if bloat > 30%)
log "Checking and reindexing bloated indexes"
psql -d $DB_NAME <<EOF
DO \$\$
DECLARE
    idx RECORD;
BEGIN
    FOR idx IN
        SELECT indexrelname
        FROM pg_stat_user_indexes
        WHERE pg_relation_size(indexrelid) > 100000000  -- > 100MB
    LOOP
        EXECUTE 'REINDEX INDEX CONCURRENTLY ' || idx.indexrelname;
        RAISE NOTICE 'Reindexed: %', idx.indexrelname;
    END LOOP;
END\$\$;
EOF

# 4. Clean up old data (if applicable)
log "Cleaning up old data"
psql -d $DB_NAME -c "DELETE FROM audit_logs WHERE created_at < NOW() - INTERVAL '90 days';"
psql -d $DB_NAME -c "DELETE FROM sessions WHERE expires_at < NOW() - INTERVAL '7 days';"

# 5. Update pg_stat_statements (reset if needed)
log "Checking pg_stat_statements"
psql -d $DB_NAME -c "SELECT pg_stat_statements_reset();" 2>/dev/null || true

log "Maintenance completed for $DB_NAME"
```

### Cron Schedule

```bash
# /etc/cron.d/db_maintenance

# Daily maintenance at 3 AM
0 3 * * * postgres /opt/scripts/maintenance.sh

# Weekly VACUUM FULL on weekends (if needed)
0 2 * * 0 postgres psql -d mydb -c "VACUUM FULL large_table;"

# Monthly full reindex
0 1 1 * * postgres psql -d mydb -c "REINDEX DATABASE mydb;"
```

## MySQL Maintenance

### OPTIMIZE TABLE

```sql
-- Defragment table and reclaim space
OPTIMIZE TABLE tablename;

-- For InnoDB (rebuilds table)
ALTER TABLE tablename ENGINE=InnoDB;

-- Check table fragmentation
SELECT
    TABLE_NAME,
    DATA_LENGTH,
    INDEX_LENGTH,
    DATA_FREE,
    ROUND(DATA_FREE / (DATA_LENGTH + INDEX_LENGTH) * 100, 2) AS fragmentation_pct
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb'
  AND DATA_FREE > 0
ORDER BY DATA_FREE DESC;
```

### ANALYZE TABLE

```sql
-- Update index statistics
ANALYZE TABLE tablename;

-- Check table statistics
SHOW TABLE STATUS LIKE 'tablename';
```

### CHECK and REPAIR

```sql
-- Check table for errors
CHECK TABLE tablename;

-- Repair table (MyISAM only)
REPAIR TABLE tablename;

-- For InnoDB, use innodb_force_recovery or backup/restore
```

## Transaction ID Wraparound Prevention

### Monitoring XID Age

```sql
-- Check tables approaching wraparound
SELECT
    c.oid::regclass AS table_name,
    age(c.relfrozenxid) AS xid_age,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS size
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(c.relfrozenxid) DESC
LIMIT 20;

-- Check database-wide XID age
SELECT
    datname,
    age(datfrozenxid) AS xid_age
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

### Preventing Wraparound

```sql
-- Force freeze on critical tables
VACUUM FREEZE tablename;

-- Aggressive freeze settings for maintenance window
SET vacuum_freeze_min_age = 0;
SET vacuum_freeze_table_age = 0;
VACUUM FREEZE;
```

## Monitoring Maintenance

### PostgreSQL Metrics

```sql
-- Autovacuum activity
SELECT
    schemaname,
    relname,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY last_autovacuum DESC NULLS LAST;

-- Currently running VACUUM
SELECT
    pid,
    datname,
    query,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS duration
FROM pg_stat_activity
WHERE query ILIKE '%vacuum%';

-- Autovacuum workers
SELECT * FROM pg_stat_progress_vacuum;
```

### Alerting Rules

```yaml
# Prometheus alerting rules
groups:
  - name: database-maintenance
    rules:
      - alert: HighTableBloat
        expr: pg_stat_user_tables_n_dead_tup > 100000
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "High dead tuple count on {{ $labels.relname }}"

      - alert: AutovacuumNotRunning
        expr: pg_stat_user_tables_last_autovacuum_time > 86400
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Autovacuum hasn't run in 24h on {{ $labels.relname }}"

      - alert: XIDWraparoundRisk
        expr: pg_database_xid_age > 500000000
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "XID wraparound risk on {{ $labels.datname }}"

      - alert: LongRunningVacuum
        expr: pg_stat_activity_vacuum_duration_seconds > 3600
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "VACUUM running for over 1 hour"
```

## Best Practices

**Do:**
- Configure autovacuum properly for workload
- Monitor dead tuple counts and bloat regularly
- Use CONCURRENTLY options when possible
- Schedule maintenance during low-traffic periods
- Test maintenance procedures in staging first
- Monitor XID age to prevent wraparound
- Use pg_repack for large table maintenance

**Don't:**
- Disable autovacuum without good reason
- Run VACUUM FULL during peak hours
- Ignore bloat warnings
- Let XID age approach wraparound threshold
- Run maintenance without monitoring impact
- Forget to update statistics after bulk operations

## Environment Variables

```bash
# Maintenance scheduling
DB_MAINTENANCE_ENABLED=true
DB_MAINTENANCE_SCHEDULE="0 3 * * *"
DB_MAINTENANCE_LOCK_TIMEOUT=3600

# Autovacuum tuning
DB_AUTOVACUUM_WORKERS=3
DB_AUTOVACUUM_NAPTIME=60
DB_AUTOVACUUM_SCALE_FACTOR=0.2

# Bloat thresholds
DB_BLOAT_WARNING_PERCENT=20
DB_BLOAT_CRITICAL_PERCENT=50

# XID monitoring
DB_XID_WARNING_AGE=150000000
DB_XID_CRITICAL_AGE=180000000
```

## Additional Resources

- [PostgreSQL VACUUM](https://www.postgresql.org/docs/current/sql-vacuum.html)
- [PostgreSQL Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)
- [pg_repack](https://github.com/reorg/pg_repack)
- [MySQL OPTIMIZE TABLE](https://dev.mysql.com/doc/refman/8.0/en/optimize-table.html)
