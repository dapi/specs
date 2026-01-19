# SRG-019 Database Performance Tuning

## Definition

Database performance tuning involves configuring database parameters, memory allocation, storage settings, and system resources to optimize database performance for specific workloads. This includes both PostgreSQL and MySQL tuning strategies.

## PostgreSQL Memory Configuration

### Shared Buffers

```ini
# postgresql.conf
# Shared memory for caching data pages
# Recommended: 25% of system RAM, up to 8GB for most workloads

# For 32GB RAM system
shared_buffers = 8GB

# For 16GB RAM system
shared_buffers = 4GB

# For 8GB RAM system
shared_buffers = 2GB
```

### Work Memory

```ini
# postgresql.conf
# Memory for sort operations, hash tables per query
# Be careful: this is per-operation, not per-connection

# Default for OLTP workloads
work_mem = 64MB

# For complex queries with many sorts/joins
work_mem = 256MB

# Calculate max memory usage:
# max_connections * work_mem * operations_per_query
```

### Maintenance Work Memory

```ini
# postgresql.conf
# Memory for VACUUM, CREATE INDEX, ALTER TABLE

# For systems with lots of maintenance operations
maintenance_work_mem = 2GB

# For smaller systems
maintenance_work_mem = 512MB

# Autovacuum workers share this
autovacuum_work_mem = 512MB
```

### Effective Cache Size

```ini
# postgresql.conf
# Hint for query planner about OS cache
# Set to ~75% of total RAM

# For 32GB RAM
effective_cache_size = 24GB

# For 16GB RAM
effective_cache_size = 12GB
```

## PostgreSQL Query Planner

### Cost Parameters

```ini
# postgresql.conf
# Adjust based on storage type

# For SSD storage
random_page_cost = 1.1
seq_page_cost = 1.0

# For HDD storage (default)
random_page_cost = 4.0
seq_page_cost = 1.0

# CPU costs (usually leave default)
cpu_tuple_cost = 0.01
cpu_index_tuple_cost = 0.005
cpu_operator_cost = 0.0025

# Parallel query thresholds
parallel_tuple_cost = 0.1
parallel_setup_cost = 1000.0
```

### Parallel Query Settings

```ini
# postgresql.conf
# Enable parallel queries for large tables

# Max parallel workers per query
max_parallel_workers_per_gather = 4

# Total parallel workers
max_parallel_workers = 8

# Minimum table size for parallel scan
min_parallel_table_scan_size = 8MB

# Minimum index size for parallel scan
min_parallel_index_scan_size = 512kB
```

## PostgreSQL WAL Configuration

### Write-Ahead Logging

```ini
# postgresql.conf

# WAL level (for replication set to 'replica' or 'logical')
wal_level = replica

# WAL buffers (default is usually fine)
wal_buffers = 64MB

# Checkpoint settings
checkpoint_completion_target = 0.9
checkpoint_timeout = 10min
max_wal_size = 4GB
min_wal_size = 1GB

# WAL compression (PostgreSQL 15+)
wal_compression = zstd

# Synchronous commit (trade durability for speed)
synchronous_commit = on  # safest
# synchronous_commit = off  # fastest but risk data loss
```

### Commit Delay

```ini
# postgresql.conf
# Group commits for high-throughput OLTP

commit_delay = 10  # microseconds
commit_siblings = 5  # min concurrent transactions
```

## PostgreSQL Connection Settings

```ini
# postgresql.conf

# Maximum connections (use connection pooler for high numbers)
max_connections = 200

# Reserved for superuser
superuser_reserved_connections = 3

# Connection timeout
tcp_keepalives_idle = 60
tcp_keepalives_interval = 10
tcp_keepalives_count = 6

# Statement timeout (prevent runaway queries)
statement_timeout = 30000  # 30 seconds

# Lock timeout
lock_timeout = 10000  # 10 seconds

# Idle transaction timeout (PostgreSQL 14+)
idle_in_transaction_session_timeout = 60000  # 1 minute
```

## MySQL/InnoDB Configuration

### Buffer Pool

```ini
# my.cnf
[mysqld]
# InnoDB buffer pool - main memory cache
# Set to 70-80% of available RAM for dedicated DB server

# For 32GB RAM
innodb_buffer_pool_size = 24G

# For 16GB RAM
innodb_buffer_pool_size = 12G

# Number of buffer pool instances (for >1GB buffer pool)
innodb_buffer_pool_instances = 8

# Buffer pool chunk size
innodb_buffer_pool_chunk_size = 128M
```

### InnoDB Log Settings

```ini
# my.cnf
[mysqld]
# Redo log size (larger = better write performance, slower recovery)
innodb_log_file_size = 2G
innodb_log_files_in_group = 2

# Log buffer size
innodb_log_buffer_size = 64M

# Flush method
innodb_flush_method = O_DIRECT

# Flush log at transaction commit
# 1 = safest (ACID compliant)
# 2 = flush to OS cache only
# 0 = flush every second (fastest, risk data loss)
innodb_flush_log_at_trx_commit = 1
```

### InnoDB I/O Settings

```ini
# my.cnf
[mysqld]
# I/O capacity (IOPS your storage can handle)
# For SSD
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000

# For HDD
innodb_io_capacity = 200
innodb_io_capacity_max = 400

# Read/write threads
innodb_read_io_threads = 4
innodb_write_io_threads = 4

# Purge threads
innodb_purge_threads = 4
```

### MySQL Connection Settings

```ini
# my.cnf
[mysqld]
# Maximum connections
max_connections = 500

# Thread cache
thread_cache_size = 50

# Connection timeout
wait_timeout = 28800
interactive_timeout = 28800

# Max allowed packet (for large queries/blobs)
max_allowed_packet = 64M
```

## OS-Level Tuning

### Linux Kernel Parameters

```bash
# /etc/sysctl.conf

# Virtual memory
vm.swappiness = 10
vm.dirty_ratio = 40
vm.dirty_background_ratio = 10

# Network
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.core.netdev_max_backlog = 65535

# Huge pages for PostgreSQL
vm.nr_hugepages = 4096  # Calculate based on shared_buffers

# File descriptors
fs.file-max = 2097152
```

### Disk I/O Scheduler

```bash
# For SSD (NVMe)
echo none > /sys/block/nvme0n1/queue/scheduler

# For SSD (SATA)
echo noop > /sys/block/sda/queue/scheduler

# Check current scheduler
cat /sys/block/sda/queue/scheduler
```

### Filesystem Mount Options

```bash
# /etc/fstab for PostgreSQL data directory
/dev/sda1 /var/lib/postgresql ext4 noatime,nodiratime,nobarrier 0 2

# For XFS (recommended for PostgreSQL)
/dev/sda1 /var/lib/postgresql xfs noatime,nodiratime,logbufs=8 0 2
```

## Workload-Specific Tuning

### OLTP Workload

```ini
# PostgreSQL for OLTP
shared_buffers = 8GB
work_mem = 64MB
effective_cache_size = 24GB
random_page_cost = 1.1
max_connections = 500
synchronous_commit = on
wal_buffers = 64MB
checkpoint_completion_target = 0.9
```

### OLAP/Analytics Workload

```ini
# PostgreSQL for OLAP
shared_buffers = 8GB
work_mem = 2GB  # Higher for complex queries
effective_cache_size = 24GB
random_page_cost = 1.1
max_connections = 50  # Fewer connections
max_parallel_workers_per_gather = 8
parallel_tuple_cost = 0.001
effective_io_concurrency = 200
```

### Mixed Workload

```ini
# PostgreSQL for mixed workload
shared_buffers = 8GB
work_mem = 256MB
effective_cache_size = 24GB
random_page_cost = 1.1
max_connections = 200
max_parallel_workers_per_gather = 4
```

## Performance Monitoring Queries

### PostgreSQL

```sql
-- Check buffer cache hit ratio
SELECT
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit)  as heap_hit,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as hit_ratio
FROM pg_statio_user_tables;

-- Check index usage
SELECT
    schemaname,
    tablename,
    indexrelname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Check table statistics
SELECT
    schemaname,
    relname,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_tup_ins,
    n_tup_upd,
    n_tup_del
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;

-- Check current settings
SELECT name, setting, unit, context
FROM pg_settings
WHERE name IN (
    'shared_buffers', 'work_mem', 'effective_cache_size',
    'random_page_cost', 'max_connections', 'wal_buffers'
);
```

### MySQL

```sql
-- Check InnoDB buffer pool status
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- Check buffer pool hit ratio
SELECT
    (1 - (
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    )) * 100 AS buffer_pool_hit_ratio;

-- Check InnoDB status
SHOW ENGINE INNODB STATUS\G

-- Check current settings
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'innodb_log_file_size';
SHOW VARIABLES LIKE 'max_connections';
```

## Automated Tuning Tools

### PGTune

```bash
# Generate PostgreSQL configuration
# https://pgtune.leopard.in.ua/

# Example output for 32GB RAM, SSD, OLTP workload:
# max_connections = 200
# shared_buffers = 8GB
# effective_cache_size = 24GB
# maintenance_work_mem = 2GB
# checkpoint_completion_target = 0.9
# wal_buffers = 64MB
# default_statistics_target = 100
# random_page_cost = 1.1
# effective_io_concurrency = 200
# work_mem = 20971kB
# min_wal_size = 1GB
# max_wal_size = 4GB
# max_worker_processes = 8
# max_parallel_workers_per_gather = 4
# max_parallel_workers = 8
```

### MySQL Tuner

```bash
# Install and run MySQLTuner
wget https://raw.githubusercontent.com/major/MySQLTuner-perl/master/mysqltuner.pl
perl mysqltuner.pl --host localhost --user root --pass password

# Review recommendations and apply carefully
```

## Best Practices

**Do:**
- Start with conservative settings and adjust based on monitoring
- Test configuration changes in staging before production
- Monitor memory usage and adjust accordingly
- Use connection pooling for high connection counts
- Set appropriate timeouts to prevent runaway queries
- Tune for your specific workload (OLTP vs OLAP)
- Keep OS and database software updated

**Don't:**
- Copy configurations from other systems without understanding
- Set work_mem too high (multiplied by connections)
- Ignore the query planner's cost estimates
- Disable synchronous_commit without understanding risks
- Tune parameters without baseline measurements
- Forget to restart database after configuration changes
- Over-allocate memory (leave room for OS cache)

## Environment Variables

```bash
# PostgreSQL memory settings
POSTGRES_SHARED_BUFFERS=8GB
POSTGRES_WORK_MEM=64MB
POSTGRES_MAINTENANCE_WORK_MEM=2GB
POSTGRES_EFFECTIVE_CACHE_SIZE=24GB

# PostgreSQL performance settings
POSTGRES_RANDOM_PAGE_COST=1.1
POSTGRES_MAX_PARALLEL_WORKERS=8
POSTGRES_MAX_CONNECTIONS=200

# MySQL/InnoDB settings
MYSQL_INNODB_BUFFER_POOL_SIZE=24G
MYSQL_INNODB_LOG_FILE_SIZE=2G
MYSQL_MAX_CONNECTIONS=500

# Workload type
DB_WORKLOAD_TYPE=oltp  # oltp|olap|mixed
```

## Additional Resources

- [PostgreSQL Performance Tips](https://www.postgresql.org/docs/current/performance-tips.html)
- [PGTune](https://pgtune.leopard.in.ua/)
- [MySQL Performance Tuning](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
- [MySQLTuner](https://github.com/major/MySQLTuner-perl)
- [Linux Performance Tuning for PostgreSQL](https://www.postgresql.org/docs/current/kernel-resources.html)
