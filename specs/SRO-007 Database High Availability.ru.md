# SRO-007 Database High Availability (Высокая доступность баз данных)

## Определение

Высокая доступность базы данных (HA) обеспечивает, что сервисы базы данных остаются доступными и работоспособными даже при аппаратных сбоях, программных проблемах или операциях обслуживания. Цель - минимизировать время простоя и потерю данных.

## Ключевые метрики

```
RTO (Recovery Time Objective): Максимально допустимое время простоя
RPO (Recovery Point Objective): Максимально допустимая потеря данных

Целевые показатели доступности:
- 99.9% (Три девятки): 8.76 часов простоя/год
- 99.99% (Четыре девятки): 52.6 минут простоя/год
- 99.999% (Пять девяток): 5.26 минут простоя/год
```

## Архитектуры HA

### Single Primary с репликами

```
        ┌─────────────────┐
        │   Приложение    │
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

**Характеристики:**
- Единая точка записи
- Множество реплик для чтения
- Автоматический failover при падении primary
- RPO: 0 (sync) до секунд (async)
- RTO: 15-60 секунд

### Multi-Primary (Active-Active)

```
        ┌─────────────────┐
        │   Приложение    │
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

**Характеристики:**
- Множество точек записи
- Требуется разрешение конфликтов
- Возможно географическое распределение
- Сложная настройка и обслуживание
- Более высокий потенциал доступности

## PostgreSQL HA с Patroni

### Архитектура

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

### Развертывание с Docker Compose

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

### Конфигурация HAProxy

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

# Primary (чтение-запись)
listen postgres_primary
    bind *:5432
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 patroni1:5432 maxconn 100 check port 8008
    server node2 patroni2:5432 maxconn 100 check port 8008
    server node3 patroni3:5432 maxconn 100 check port 8008

# Реплики (только чтение)
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

## MySQL HA с Group Replication

### Конфигурация

```ini
# my.cnf для каждого узла
[mysqld]
server_id = 1  # Уникальный для каждого узла
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

# Single-primary режим (по умолчанию)
group_replication_single_primary_mode = ON
group_replication_enforce_update_everywhere_checks = OFF
```

### Инициализация группы

```sql
-- Только на первом узле
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;

-- На остальных узлах
START GROUP_REPLICATION;

-- Проверить статус
SELECT * FROM performance_schema.replication_group_members;
```

## Процедуры Failover

### Автоматический Failover (Patroni)

```bash
# Проверить статус кластера
patronictl -c /etc/patroni.yml list

# Принудительный failover на конкретный узел
patronictl -c /etc/patroni.yml failover --candidate node2

# Switchover (плановый)
patronictl -c /etc/patroni.yml switchover --master node1 --candidate node2

# Реинициализировать упавший узел
patronictl -c /etc/patroni.yml reinit postgres-cluster node1
```

### Шаги ручного Failover

```bash
# 1. Проверить что primary недоступен
pg_isready -h primary -p 5432

# 2. Проверить лаг реплики
psql -h replica -c "SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();"

# 3. Промоутить реплику
pg_ctl promote -D /var/lib/postgresql/data
# Или: SELECT pg_promote();

# 4. Обновить DNS/Load balancer
# 5. Проверить что новый primary принимает записи
# 6. Обновить connection strings приложения при необходимости
# 7. Перенастроить старый primary как реплику
```

## Проверки здоровья

### Скрипт проверки здоровья PostgreSQL

```bash
#!/bin/bash
# health_check.sh

HOST=${1:-localhost}
PORT=${2:-5432}
USER=${3:-postgres}

# Проверить что PostgreSQL принимает соединения
if ! pg_isready -h $HOST -p $PORT -U $USER > /dev/null 2>&1; then
    echo "PostgreSQL не готов"
    exit 1
fi

# Проверить primary это или replica
IS_RECOVERY=$(psql -h $HOST -p $PORT -U $USER -t -c "SELECT pg_is_in_recovery();" 2>/dev/null | tr -d ' ')

if [ "$IS_RECOVERY" = "f" ]; then
    echo "Primary"
    exit 0
elif [ "$IS_RECOVERY" = "t" ]; then
    echo "Replica"
    exit 0
else
    echo "Неизвестное состояние"
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

## Мониторинг и алертинг

```yaml
# Правила алертинга Prometheus
groups:
  - name: postgres-ha
    rules:
      - alert: PostgresPrimaryDown
        expr: pg_up{role="primary"} == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL primary недоступен"

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

## Best practices

**Делать:**
- Использовать минимум 3 узла для quorum-based HA
- Настраивать синхронную репликацию для нулевой потери данных
- Регулярно тестировать процедуры failover
- Непрерывно мониторить здоровье кластера
- Использовать connection pooling (PgBouncer)
- Реализовать proper fencing для предотвращения split-brain
- Документировать runbooks для всех сценариев сбоев
- Поддерживать etcd/consensus кластер здоровым

**Не делать:**
- Запускать consensus кластер на тех же узлах что и БД
- Игнорировать предупреждения о лаге репликации
- Пропускать тестирование failover
- Использовать HA без должного мониторинга
- Забывать тестировать переподключение приложения
- Пренебрегать стратегией бэкапов (HA это не бэкап)
- Использовать синхронную репликацию через каналы с высокой задержкой

## Переменные окружения

```bash
# Patroni
PATRONI_SCOPE=postgres-cluster
PATRONI_NAME=node1
PATRONI_ETCD3_HOSTS=etcd1:2379,etcd2:2379,etcd3:2379
PATRONI_POSTGRESQL_DATA_DIR=/var/lib/postgresql/data

# Настройки failover
DB_HA_FAILOVER_TIMEOUT=30
DB_HA_MAX_LAG_ON_FAILOVER=1048576
DB_HA_SYNCHRONOUS_MODE=on

# Проверка здоровья
DB_HEALTH_CHECK_INTERVAL=3
DB_HEALTH_CHECK_TIMEOUT=10
```

## Дополнительные ресурсы

- [Patroni Documentation](https://patroni.readthedocs.io/)
- [PostgreSQL High Availability](https://www.postgresql.org/docs/current/high-availability.html)
- [MySQL Group Replication](https://dev.mysql.com/doc/refman/8.0/en/group-replication.html)
- [etcd Documentation](https://etcd.io/docs/)
- [HAProxy Configuration](https://www.haproxy.org/download/2.6/doc/configuration.txt)
