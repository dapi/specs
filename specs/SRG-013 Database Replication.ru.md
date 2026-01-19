# SRG-013 Database Replication (Репликация баз данных)

## Определение

Репликация базы данных - это процесс копирования и поддержания объектов базы данных в нескольких базах данных, составляющих распределенную систему. Она обеспечивает избыточность данных, повышает доступность и позволяет масштабировать чтение.

## Типы репликации

### Синхронная репликация

```
Клиент → Primary → Replica (ждем ACK) → Клиент ACK

Временная шкала:
1. Клиент отправляет запись на Primary
2. Primary записывает в WAL
3. Primary отправляет WAL на Replica
4. Replica записывает в WAL и отправляет ACK
5. Primary коммитит и отвечает клиенту

Задержка: +2-10ms на запись (сетевой round-trip)
```

**Случаи использования:**
- Финансовые транзакции
- Критичные данные, которые нельзя потерять
- Требования комплаенса (нулевая потеря данных)

### Асинхронная репликация

```
Клиент → Primary → Клиент ACK
              ↓ (async)
           Replica

Временная шкала:
1. Клиент отправляет запись на Primary
2. Primary записывает в WAL и коммитит
3. Primary отвечает клиенту
4. (Фоново) Primary отправляет WAL на Replica

Задержка: Нет дополнительной задержки
Лаг: 0-N секунд (обычно <1с)
```

**Случаи использования:**
- Масштабирование чтения
- Географическое распределение
- Аналитические нагрузки
- Резервные копии

### Полусинхронная репликация

```
Клиент → Primary → Минимум 1 Replica ACK → Клиент ACK

Гарантии: Минимум одна реплика имеет данные перед коммитом
```

## PostgreSQL Streaming Replication

### Конфигурация Primary

```bash
# postgresql.conf
wal_level = replica                    # Включить репликацию
max_wal_senders = 10                   # Макс количество реплик
wal_keep_size = 1GB                    # Хранение WAL для реплик
hot_standby = on                       # Разрешить запросы на реплике

# Синхронная репликация (опционально)
synchronous_commit = on                # Ждать ACK от реплики
synchronous_standby_names = 'replica1' # Имена реплик
```

```bash
# pg_hba.conf - Разрешить репликационные соединения
host    replication     replicator    10.0.0.0/8    scram-sha-256
```

### Конфигурация Replica

```bash
# postgresql.conf
hot_standby = on                       # Разрешить read запросы
primary_conninfo = 'host=primary port=5432 user=replicator password=secret'
primary_slot_name = 'replica1_slot'    # Репликационный слот
```

```bash
# Создать файл standby signal
touch /var/lib/postgresql/data/standby.signal
```

### Репликационные слоты

Предотвращают удаление WAL до получения репликой:

```sql
-- На Primary: Создать слот
SELECT pg_create_physical_replication_slot('replica1_slot');

-- Список слотов
SELECT slot_name, active, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots;

-- Удалить слот (если реплика удалена)
SELECT pg_drop_replication_slot('replica1_slot');
```

### Мониторинг лага репликации

```sql
-- На Primary: Проверить статус репликации
SELECT client_addr,
       state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) / 1024 / 1024 AS lag_mb
FROM pg_stat_replication;

-- На Replica: Проверить лаг в секундах
SELECT CASE
    WHEN pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn() THEN 0
    ELSE EXTRACT(EPOCH FROM now() - pg_last_xact_replay_timestamp())
END AS lag_seconds;
```

## MySQL Репликация

### GTID-Based репликация (рекомендуется)

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

### Настройка Replica

```sql
-- На Replica
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST = 'primary',
    SOURCE_PORT = 3306,
    SOURCE_USER = 'replicator',
    SOURCE_PASSWORD = 'secret',
    SOURCE_AUTO_POSITION = 1;

START REPLICA;
SHOW REPLICA STATUS\G
```

### Мониторинг MySQL репликации

```sql
-- Проверить статус репликации
SHOW REPLICA STATUS\G

-- Ключевые поля для мониторинга:
-- Replica_IO_Running: Yes
-- Replica_SQL_Running: Yes
-- Seconds_Behind_Source: 0
-- Last_Error: (пусто)
```

## Логическая репликация (PostgreSQL)

Для выборочной репликации таблиц:

```sql
-- На Primary: Создать публикацию
CREATE PUBLICATION my_pub FOR TABLE users, orders;

-- На Replica: Создать подписку
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=primary dbname=mydb user=replicator'
    PUBLICATION my_pub;

-- Мониторинг
SELECT * FROM pg_stat_subscription;
```

## Стратегии Failover

### Ручной Failover

```bash
# 1. Проверить что primary недоступен
pg_isready -h primary -p 5432

# 2. Промоутить реплику в primary
pg_ctl promote -D /var/lib/postgresql/data

# Или через SQL
SELECT pg_promote();

# 3. Обновить connection strings в приложении
# 4. Перенастроить старый primary как реплику (если восстановим)
```

### Автоматический Failover с Patroni

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
    maximum_lag_on_failover: 1048576  # 1MB макс лаг для failover
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

### Метрики Failover

```
# Целевые метрики
RTO (Recovery Time Objective): < 30 секунд
RPO (Recovery Point Objective): 0 (sync) или < 1 секунды (async)

# Временная шкала failover в Patroni
1. Обнаружение падения лидера: 10-30 секунд (TTL)
2. Выбор нового лидера: 1-5 секунд
3. Перенаправление соединений: 1-10 секунд
Итого: 15-45 секунд типично
```

## Маршрутизация соединений

### PgBouncer с HAProxy

```bash
# Конфигурация HAProxy
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

### Маршрутизация на уровне приложения

```python
# Python с psycopg
import psycopg

# Запись на primary
write_conn = psycopg.connect(
    "host=primary port=5432 dbname=mydb",
    target_session_attrs="read-write"
)

# Чтение с реплики
read_conn = psycopg.connect(
    "host=primary,replica1,replica2 port=5432 dbname=mydb",
    target_session_attrs="prefer-standby"
)
```

## Метрики и мониторинг

```
# Здоровье репликации
db_replication_lag_bytes (gauge)       # Лаг в байтах
db_replication_lag_seconds (gauge)     # Лаг в секундах
db_replication_state (gauge)           # 0=отключен, 1=streaming
db_replication_slots_active (gauge)    # Активные репликационные слоты

# Метрики WAL
db_wal_segments_count (gauge)          # Количество WAL сегментов
db_wal_write_rate_bytes (counter)      # Скорость записи WAL
db_wal_sent_bytes (counter)            # WAL отправлено на реплики

# Метрики failover
db_failover_total (counter)            # Количество failover
db_failover_duration_seconds (histogram) # Длительность failover
```

### Правила алертинга

```yaml
# Правила алертинга Prometheus
groups:
  - name: replication
    rules:
      - alert: ReplicationLagHigh
        expr: db_replication_lag_seconds > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Высокий лаг репликации"

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

## Best practices

**Делать:**
- Использовать синхронную репликацию для критичных данных
- Настраивать репликационные слоты для предотвращения потери WAL
- Непрерывно мониторить лаг репликации
- Регулярно тестировать процедуры failover
- Использовать connection pooling с правильной маршрутизацией
- Держать реплики близко к primary (низкая задержка)
- Документировать runbooks для failover
- Использовать GTID для MySQL репликации

**Не делать:**
- Игнорировать алерты о лаге репликации
- Использовать синхронную репликацию между регионами (высокая задержка)
- Забывать очищать неиспользуемые репликационные слоты
- Пропускать тестирование failover
- Выполнять тяжелые запросы на primary когда доступны реплики
- Использовать логическую репликацию для высоконагруженных таблиц без тестирования

## Конфигурация окружения

```bash
# Primary
DB_REPLICATION_ROLE=primary
DB_REPLICATION_MODE=async              # sync|async|semi-sync
DB_WAL_LEVEL=replica
DB_MAX_WAL_SENDERS=10
DB_WAL_KEEP_SIZE=2GB
DB_SYNC_STANDBY_NAMES=                 # Пусто для async

# Replica
DB_REPLICATION_ROLE=replica
DB_PRIMARY_HOST=primary
DB_PRIMARY_PORT=5432
DB_REPLICATION_SLOT=replica1_slot
DB_HOT_STANDBY=on
```

## Дополнительные ресурсы

- [PostgreSQL Streaming Replication](https://www.postgresql.org/docs/current/warm-standby.html)
- [MySQL Replication](https://dev.mysql.com/doc/refman/8.0/en/replication.html)
- [Patroni Documentation](https://patroni.readthedocs.io/)
- [PgBouncer](https://www.pgbouncer.org/)
