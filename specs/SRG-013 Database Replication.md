# SRG-013 Database Replication

## Definition

Database Replication is the process of copying and maintaining database objects in multiple databases that make up a distributed database system. It provides data redundancy, improves availability, and enables read scaling.

## Replication Types

### Synchronous Replication

```
Client → Primary → Replica (wait for ACK) → Client ACK

Timeline:
1. Client sends write to Primary
2. Primary writes to WAL
3. Primary sends WAL to Replica
4. Replica writes to WAL and sends ACK
5. Primary commits and responds to Client

Latency: +2-10ms per write (network round-trip)
```

**Use cases:**
- Financial transactions
- Critical data that cannot be lost
- Compliance requirements (zero data loss)

### Asynchronous Replication

```
Client → Primary → Client ACK
              ↓ (async)
           Replica

Timeline:
1. Client sends write to Primary
2. Primary writes to WAL and commits
3. Primary responds to Client
4. (Background) Primary sends WAL to Replica

Latency: No additional latency
Lag: 0-N seconds (typically <1s)
```

**Use cases:**
- Read scaling
- Geographic distribution
- Analytics workloads
- Backup destinations

### Semi-Synchronous Replication

```
Client → Primary → At least 1 Replica ACK → Client ACK

Guarantees: At least one replica has the data before commit
```

## PostgreSQL Streaming Replication

### Primary Configuration

```bash
# postgresql.conf
wal_level = replica                    # Enable replication
max_wal_senders = 10                   # Max number of replicas
wal_keep_size = 1GB                    # WAL retention for replicas
hot_standby = on                       # Allow queries on replica

# Synchronous replication (optional)
synchronous_commit = on                # Wait for replica ACK
synchronous_standby_names = 'replica1' # Replica names
```

```bash
# pg_hba.conf - Allow replication connections
host    replication     replicator    10.0.0.0/8    scram-sha-256
```

### Replica Configuration

```bash
# postgresql.conf
hot_standby = on                       # Allow read queries
primary_conninfo = 'host=primary port=5432 user=replicator password=secret'
primary_slot_name = 'replica1_slot'    # Replication slot
```

```bash
# Create standby signal file
touch /var/lib/postgresql/data/standby.signal
```

### Replication Slots

Prevent WAL deletion before replica receives it:

```sql
-- On Primary: Create slot
SELECT pg_create_physical_replication_slot('replica1_slot');

-- List slots
SELECT slot_name, active, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots;

-- Drop slot (if replica removed)
SELECT pg_drop_replication_slot('replica1_slot');
```

### Monitoring Replication Lag

```sql
-- On Primary: Check replication status
SELECT client_addr,
       state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) / 1024 / 1024 AS lag_mb
FROM pg_stat_replication;

-- On Replica: Check lag in seconds
SELECT CASE
    WHEN pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn() THEN 0
    ELSE EXTRACT(EPOCH FROM now() - pg_last_xact_replay_timestamp())
END AS lag_seconds;
```

## MySQL Replication

### GTID-Based Replication (Recommended)

```sql
-- Primary my.cnf
[mysqld]
server-id = 1
log_bin = mysql-bin
gtid_mode = ON
enforce_gtid_consistency = ON
binlog_format = ROW
sync_binlog = 1

-- Replica my.cnf
[mysqld]
server-id = 2
relay_log = relay-bin
gtid_mode = ON
enforce_gtid_consistency = ON
read_only = ON
super_read_only = ON
```

### Setup Replica

```sql
-- On Replica
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST = 'primary',
    SOURCE_PORT = 3306,
    SOURCE_USER = 'replicator',
    SOURCE_PASSWORD = 'secret',
    SOURCE_AUTO_POSITION = 1;

START REPLICA;
SHOW REPLICA STATUS\G
```

### Monitoring MySQL Replication

```sql
-- Check replication status
SHOW REPLICA STATUS\G

-- Key fields to monitor:
-- Replica_IO_Running: Yes
-- Replica_SQL_Running: Yes
-- Seconds_Behind_Source: 0
-- Last_Error: (empty)
```

## Logical Replication (PostgreSQL)

For selective table replication:

```sql
-- On Primary: Create publication
CREATE PUBLICATION my_pub FOR TABLE users, orders;

-- On Replica: Create subscription
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=primary dbname=mydb user=replicator'
    PUBLICATION my_pub;

-- Monitor
SELECT * FROM pg_stat_subscription;
```

## Failover Strategies

### Manual Failover

```bash
# 1. Verify primary is down
pg_isready -h primary -p 5432

# 2. Promote replica to primary
pg_ctl promote -D /var/lib/postgresql/data

# Or via SQL
SELECT pg_promote();

# 3. Update application connection strings
# 4. Reconfigure old primary as replica (if recoverable)
```

### Automatic Failover with Patroni

```yaml
# patroni.yml
scope: postgres-cluster
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: node1:8008

etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1MB max lag for failover
    postgresql:
      use_pg_rewind: true
      parameters:
        wal_level: replica
        max_wal_senders: 10
        max_replication_slots: 10

postgresql:
  listen: 0.0.0.0:5432
  connect_address: node1:5432
  data_dir: /var/lib/postgresql/data
  authentication:
    replication:
      username: replicator
      password: secret
```

### Failover Metrics

```
# Target metrics
RTO (Recovery Time Objective): < 30 seconds
RPO (Recovery Point Objective): 0 (sync) or < 1 second (async)

# Patroni failover timeline
1. Leader failure detected: 10-30 seconds (TTL)
2. New leader elected: 1-5 seconds
3. Connections redirected: 1-10 seconds
Total: 15-45 seconds typical
```

## Connection Routing

### PgBouncer with HAProxy

```bash
# HAProxy configuration
frontend postgres_frontend
    bind *:5432
    default_backend postgres_primary

backend postgres_primary
    option httpchk GET /primary
    http-check expect status 200
    server node1 node1:5432 check port 8008
    server node2 node2:5432 check port 8008
    server node3 node3:5432 check port 8008

backend postgres_replicas
    balance roundrobin
    option httpchk GET /replica
    http-check expect status 200
    server node1 node1:5432 check port 8008
    server node2 node2:5432 check port 8008
    server node3 node3:5432 check port 8008
```

### Application-Level Routing

```python
# Python with psycopg
import psycopg

# Write to primary
write_conn = psycopg.connect(
    "host=primary port=5432 dbname=mydb",
    target_session_attrs="read-write"
)

# Read from replica
read_conn = psycopg.connect(
    "host=primary,replica1,replica2 port=5432 dbname=mydb",
    target_session_attrs="prefer-standby"
)
```

## Metrics and Monitoring

```
# Replication health
db_replication_lag_bytes (gauge)       # Lag in bytes
db_replication_lag_seconds (gauge)     # Lag in seconds
db_replication_state (gauge)           # 0=disconnected, 1=streaming
db_replication_slots_active (gauge)    # Active replication slots

# WAL metrics
db_wal_segments_count (gauge)          # Number of WAL segments
db_wal_write_rate_bytes (counter)      # WAL write rate
db_wal_sent_bytes (counter)            # WAL sent to replicas

# Failover metrics
db_failover_total (counter)            # Number of failovers
db_failover_duration_seconds (histogram) # Failover duration
```

### Alerting Rules

```yaml
# Prometheus alerting rules
groups:
  - name: replication
    rules:
      - alert: ReplicationLagHigh
        expr: db_replication_lag_seconds > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Replication lag is high"

      - alert: ReplicationLagCritical
        expr: db_replication_lag_seconds > 300
        for: 1m
        labels:
          severity: critical

      - alert: ReplicationDown
        expr: db_replication_state == 0
        for: 1m
        labels:
          severity: critical

      - alert: ReplicationSlotInactive
        expr: db_replication_slots_active == 0
        for: 5m
        labels:
          severity: warning
```

## Best Practices

**Do:**
- Use synchronous replication for critical data
- Configure replication slots to prevent WAL loss
- Monitor replication lag continuously
- Test failover procedures regularly
- Use connection pooling with proper routing
- Keep replicas close to primary (low latency)
- Document failover runbooks
- Use GTID for MySQL replication

**Don't:**
- Ignore replication lag alerts
- Use synchronous replication across regions (high latency)
- Forget to clean up unused replication slots
- Skip failover testing
- Run heavy queries on primary when replicas available
- Use logical replication for high-throughput tables without testing

## Environment Configuration

```bash
# Primary
DB_REPLICATION_ROLE=primary
DB_REPLICATION_MODE=async              # sync|async|semi-sync
DB_WAL_LEVEL=replica
DB_MAX_WAL_SENDERS=10
DB_WAL_KEEP_SIZE=2GB
DB_SYNC_STANDBY_NAMES=                 # Empty for async

# Replica
DB_REPLICATION_ROLE=replica
DB_PRIMARY_HOST=primary
DB_PRIMARY_PORT=5432
DB_REPLICATION_SLOT=replica1_slot
DB_HOT_STANDBY=on
```

## Additional Resources

- [PostgreSQL Streaming Replication](https://www.postgresql.org/docs/current/warm-standby.html)
- [MySQL Replication](https://dev.mysql.com/doc/refman/8.0/en/replication.html)
- [Patroni Documentation](https://patroni.readthedocs.io/)
- [PgBouncer](https://www.pgbouncer.org/)
