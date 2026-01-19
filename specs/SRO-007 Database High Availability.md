# SRO-007 Database High Availability

## Definition

Database High Availability (HA) ensures that database services remain accessible and operational even during hardware failures, software issues, or maintenance operations. The goal is to minimize downtime and data loss.

## Key Metrics

```
RTO (Recovery Time Objective): Maximum acceptable downtime
RPO (Recovery Point Objective): Maximum acceptable data loss

Availability Targets:
- 99.9% (Three 9s): 8.76 hours downtime/year
- 99.99% (Four 9s): 52.6 minutes downtime/year
- 99.999% (Five 9s): 5.26 minutes downtime/year
```

## HA Architectures

### Single Primary with Replicas

```
        ┌─────────────────┐
        │   Application   │
        └────────┬────────┘
                 │
        ┌────────▼────────┐
        │   HAProxy/LB    │
        └────────┬────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───▼───┐   ┌───▼───┐   ┌───▼───┐
│Primary│   │Replica│   │Replica│
│  RW   │──▶│  RO   │──▶│  RO   │
└───────┘   └───────┘   └───────┘
```

**Characteristics:**
- Single write point
- Multiple read replicas
- Automatic failover on primary failure
- RPO: 0 (sync) to seconds (async)
- RTO: 15-60 seconds

### Multi-Primary (Active-Active)

```
        ┌─────────────────┐
        │   Application   │
        └────────┬────────┘
                 │
        ┌────────▼────────┐
        │   HAProxy/LB    │
        └────────┬────────┘
                 │
    ┌────────────┴────────────┐
    │                         │
┌───▼───┐               ┌───▼───┐
│Primary│◀─────────────▶│Primary│
│  RW   │   Sync/BDR    │  RW   │
└───────┘               └───────┘
```

**Characteristics:**
- Multiple write points
- Conflict resolution required
- Geographic distribution possible
- Complex setup and maintenance
- Higher availability potential

## PostgreSQL HA with Patroni

### Architecture

```yaml
# patroni.yml
scope: postgres-cluster
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: node1:8008

etcd3:
  hosts:
    - etcd1:2379
    - etcd2:2379
    - etcd3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_connections: 200
        max_wal_senders: 10
        max_replication_slots: 10
        wal_keep_size: 1GB
        synchronous_commit: "on"
        synchronous_standby_names: "*"

  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: node1:5432
  data_dir: /var/lib/postgresql/data
  bin_dir: /usr/lib/postgresql/15/bin
  authentication:
    replication:
      username: replicator
      password: rep_password
    superuser:
      username: postgres
      password: postgres_password

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

### Deployment with Docker Compose

```yaml
version: '3.8'

services:
  etcd1:
    image: quay.io/coreos/etcd:v3.5.0
    command:
      - etcd
      - --name=etcd1
      - --initial-advertise-peer-urls=http://etcd1:2380
      - --listen-peer-urls=http://0.0.0.0:2380
      - --listen-client-urls=http://0.0.0.0:2379
      - --advertise-client-urls=http://etcd1:2379
      - --initial-cluster=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      - --initial-cluster-state=new
    networks:
      - postgres-ha

  patroni1:
    image: patroni/patroni:latest
    environment:
      PATRONI_NAME: node1
      PATRONI_SCOPE: postgres-cluster
      PATRONI_ETCD3_HOSTS: etcd1:2379,etcd2:2379,etcd3:2379
      PATRONI_RESTAPI_LISTEN: 0.0.0.0:8008
      PATRONI_RESTAPI_CONNECT_ADDRESS: patroni1:8008
      PATRONI_POSTGRESQL_LISTEN: 0.0.0.0:5432
      PATRONI_POSTGRESQL_CONNECT_ADDRESS: patroni1:5432
      PATRONI_POSTGRESQL_DATA_DIR: /var/lib/postgresql/data
      PATRONI_REPLICATION_USERNAME: replicator
      PATRONI_REPLICATION_PASSWORD: rep_password
      PATRONI_SUPERUSER_USERNAME: postgres
      PATRONI_SUPERUSER_PASSWORD: postgres_password
    volumes:
      - patroni1_data:/var/lib/postgresql/data
    networks:
      - postgres-ha

  haproxy:
    image: haproxy:2.6
    ports:
      - "5432:5432"
      - "5433:5433"
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    networks:
      - postgres-ha

networks:
  postgres-ha:
    driver: bridge

volumes:
  patroni1_data:
  patroni2_data:
  patroni3_data:
```

### HAProxy Configuration

```bash
# haproxy.cfg
global
    maxconn 1000

defaults
    mode tcp
    timeout connect 10s
    timeout client 30s
    timeout server 30s

listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats

# Primary (read-write)
listen postgres_primary
    bind *:5432
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 patroni1:5432 maxconn 100 check port 8008
    server node2 patroni2:5432 maxconn 100 check port 8008
    server node3 patroni3:5432 maxconn 100 check port 8008

# Replicas (read-only)
listen postgres_replicas
    bind *:5433
    balance roundrobin
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 patroni1:5432 maxconn 100 check port 8008
    server node2 patroni2:5432 maxconn 100 check port 8008
    server node3 patroni3:5432 maxconn 100 check port 8008
```

## MySQL HA with Group Replication

### Configuration

```ini
# my.cnf for each node
[mysqld]
server_id = 1  # Unique per node
gtid_mode = ON
enforce_gtid_consistency = ON
binlog_checksum = NONE
log_bin = binlog
log_slave_updates = ON
binlog_format = ROW
master_info_repository = TABLE
relay_log_info_repository = TABLE
transaction_write_set_extraction = XXHASH64

# Group Replication
plugin_load_add = 'group_replication.so'
group_replication_group_name = "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
group_replication_start_on_boot = OFF
group_replication_local_address = "node1:33061"
group_replication_group_seeds = "node1:33061,node2:33061,node3:33061"
group_replication_bootstrap_group = OFF

# Single-primary mode (default)
group_replication_single_primary_mode = ON
group_replication_enforce_update_everywhere_checks = OFF
```

### Bootstrap Group

```sql
-- On first node only
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;

-- On other nodes
START GROUP_REPLICATION;

-- Check status
SELECT * FROM performance_schema.replication_group_members;
```

## Failover Procedures

### Automatic Failover (Patroni)

```bash
# Check cluster status
patronictl -c /etc/patroni.yml list

# Force failover to specific node
patronictl -c /etc/patroni.yml failover --candidate node2

# Switchover (planned)
patronictl -c /etc/patroni.yml switchover --master node1 --candidate node2

# Reinitialize failed node
patronictl -c /etc/patroni.yml reinit postgres-cluster node1
```

### Manual Failover Steps

```bash
# 1. Verify primary is down
pg_isready -h primary -p 5432

# 2. Check replica lag
psql -h replica -c "SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();"

# 3. Promote replica
pg_ctl promote -D /var/lib/postgresql/data
# Or: SELECT pg_promote();

# 4. Update DNS/Load balancer
# 5. Verify new primary accepts writes
# 6. Update application connection strings if needed
# 7. Reconfigure old primary as replica
```

## Health Checks

### PostgreSQL Health Check Script

```bash
#!/bin/bash
# health_check.sh

HOST=${1:-localhost}
PORT=${2:-5432}
USER=${3:-postgres}

# Check if PostgreSQL is accepting connections
if ! pg_isready -h $HOST -p $PORT -U $USER > /dev/null 2>&1; then
    echo "PostgreSQL is not ready"
    exit 1
fi

# Check if it's primary or replica
IS_RECOVERY=$(psql -h $HOST -p $PORT -U $USER -t -c "SELECT pg_is_in_recovery();" 2>/dev/null | tr -d ' ')

if [ "$IS_RECOVERY" = "f" ]; then
    echo "Primary"
    exit 0
elif [ "$IS_RECOVERY" = "t" ]; then
    echo "Replica"
    exit 0
else
    echo "Unknown state"
    exit 1
fi
```

### Kubernetes Probes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  containers:
    - name: postgres
      image: postgres:15
      livenessProbe:
        exec:
          command:
            - pg_isready
            - -U
            - postgres
        initialDelaySeconds: 30
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 3
      readinessProbe:
        exec:
          command:
            - psql
            - -U
            - postgres
            - -c
            - SELECT 1
        initialDelaySeconds: 5
        periodSeconds: 5
        timeoutSeconds: 3
        failureThreshold: 3
```

## Monitoring and Alerting

```yaml
# Prometheus alerting rules
groups:
  - name: postgres-ha
    rules:
      - alert: PostgresPrimaryDown
        expr: pg_up{role="primary"} == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL primary is down"

      - alert: PostgresReplicaLagHigh
        expr: pg_replication_lag > 30
        for: 5m
        labels:
          severity: warning

      - alert: PatroniClusterUnhealthy
        expr: patroni_cluster_unlocked == 1
        for: 1m
        labels:
          severity: critical

      - alert: PatroniNoLeader
        expr: sum(patroni_master) == 0
        for: 30s
        labels:
          severity: critical

      - alert: EtcdClusterUnhealthy
        expr: etcd_server_has_leader == 0
        for: 30s
        labels:
          severity: critical
```

## Best Practices

**Do:**
- Use at least 3 nodes for quorum-based HA
- Configure synchronous replication for zero data loss
- Test failover procedures regularly
- Monitor cluster health continuously
- Use connection pooling (PgBouncer)
- Implement proper fencing to prevent split-brain
- Document runbooks for all failure scenarios
- Keep etcd/consensus cluster healthy

**Don't:**
- Run consensus cluster on same nodes as database
- Ignore replication lag warnings
- Skip failover testing
- Use HA without proper monitoring
- Forget to test application reconnection
- Neglect backup strategy (HA is not backup)
- Use synchronous replication across high-latency links

## Environment Variables

```bash
# Patroni
PATRONI_SCOPE=postgres-cluster
PATRONI_NAME=node1
PATRONI_ETCD3_HOSTS=etcd1:2379,etcd2:2379,etcd3:2379
PATRONI_POSTGRESQL_DATA_DIR=/var/lib/postgresql/data

# Failover settings
DB_HA_FAILOVER_TIMEOUT=30
DB_HA_MAX_LAG_ON_FAILOVER=1048576
DB_HA_SYNCHRONOUS_MODE=on

# Health check
DB_HEALTH_CHECK_INTERVAL=3
DB_HEALTH_CHECK_TIMEOUT=10
```

## Additional Resources

- [Patroni Documentation](https://patroni.readthedocs.io/)
- [PostgreSQL High Availability](https://www.postgresql.org/docs/current/high-availability.html)
- [MySQL Group Replication](https://dev.mysql.com/doc/refman/8.0/en/group-replication.html)
- [etcd Documentation](https://etcd.io/docs/)
- [HAProxy Configuration](https://www.haproxy.org/download/2.6/doc/configuration.txt)
