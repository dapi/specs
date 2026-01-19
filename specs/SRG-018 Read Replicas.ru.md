# SRG-018 Read Replicas & Load Balancing (Реплики чтения и балансировка нагрузки)

## Определение

Реплики чтения - это копии базы данных, обслуживающие read-only запросы, распределяя нагрузку чтения между несколькими экземплярами, пока primary обрабатывает записи. Этот паттерн позволяет горизонтально масштабировать чтение без сложности шардирования.

## Архитектура

```
                    ┌─────────────┐
                    │ Приложение  │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
       ┌──────▼──────┐          ┌──────▼──────┐
       │   Записи   │          │   Чтение    │
       │ (Primary)   │          │(Load Balancer)│
       └──────┬──────┘          └──────┬──────┘
              │                        │
              │              ┌─────────┼─────────┐
              │              │         │         │
              ▼              ▼         ▼         ▼
        ┌─────────┐    ┌─────────┐┌─────────┐┌─────────┐
        │ Primary │───▶│Replica 1││Replica 2││Replica 3│
        │   RW    │    │   RO    ││   RO    ││   RO    │
        └─────────┘    └─────────┘└─────────┘└─────────┘
```

## Стратегии балансировки нагрузки

### Round Robin

```python
class RoundRobinBalancer:
    def __init__(self, replicas):
        self.replicas = replicas
        self.current = 0

    def get_replica(self):
        replica = self.replicas[self.current]
        self.current = (self.current + 1) % len(self.replicas)
        return replica
```

### Взвешенное распределение

```python
class WeightedBalancer:
    def __init__(self, replicas_with_weights):
        # [('replica1', 3), ('replica2', 2), ('replica3', 1)]
        self.replicas = []
        for replica, weight in replicas_with_weights:
            self.replicas.extend([replica] * weight)

    def get_replica(self):
        return random.choice(self.replicas)
```

### Least Connections

```python
class LeastConnectionsBalancer:
    def __init__(self, replicas):
        self.connections = {r: 0 for r in replicas}

    def get_replica(self):
        return min(self.connections, key=self.connections.get)

    def release(self, replica):
        self.connections[replica] -= 1
```

### Lag-Aware маршрутизация

```python
class LagAwareBalancer:
    def __init__(self, replicas, max_lag_seconds=5):
        self.replicas = replicas
        self.max_lag = max_lag_seconds

    def get_replica(self):
        available = [r for r in self.replicas if self.get_lag(r) < self.max_lag]
        if not available:
            raise Exception("Нет реплик с приемлемым лагом")
        return random.choice(available)

    def get_lag(self, replica):
        # Запросить реплику о лаге репликации
        return query_lag(replica)
```

## Реализация PostgreSQL

### Connection String с несколькими хостами

```python
# psycopg3 - автоматический failover
import psycopg

# Чтение с реплик, failover на primary
read_conn = psycopg.connect(
    "host=replica1,replica2,replica3,primary "
    "port=5432 "
    "dbname=mydb "
    "target_session_attrs=prefer-standby"
)

# Запись только на primary
write_conn = psycopg.connect(
    "host=primary,replica1,replica2 "
    "port=5432 "
    "dbname=mydb "
    "target_session_attrs=read-write"
)
```

### Конфигурация HAProxy

```bash
# haproxy.cfg
frontend postgres_read
    bind *:5433
    default_backend postgres_replicas

backend postgres_replicas
    balance roundrobin
    option httpchk GET /replica
    http-check expect status 200
    server replica1 replica1:5432 check port 8008 weight 3
    server replica2 replica2:5432 check port 8008 weight 2
    server replica3 replica3:5432 check port 8008 weight 1
```

### Конфигурация PgBouncer

```ini
# pgbouncer.ini
[databases]
mydb_write = host=primary port=5432 dbname=mydb
mydb_read = host=replica1,replica2,replica3 port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
```

## Разделение чтения/записи на уровне приложения

### Rails с ActiveRecord

```ruby
# config/database.yml
production:
  primary:
    adapter: postgresql
    host: primary.db.local
    database: mydb
  primary_replica:
    adapter: postgresql
    host: replica.db.local
    database: mydb
    replica: true

# В модели
class ApplicationRecord < ActiveRecord::Base
  connects_to database: { writing: :primary, reading: :primary_replica }
end

# Автоматическое переключение
ActiveRecord::Base.connected_to(role: :reading) do
  User.find(1)  # Использует реплику
end

# Явная запись
ActiveRecord::Base.connected_to(role: :writing) do
  User.create!(name: 'John')  # Использует primary
end
```

### Django с Database Routers

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'primary.db.local',
        'NAME': 'mydb',
    },
    'replica': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'replica.db.local',
        'NAME': 'mydb',
    }
}

DATABASE_ROUTERS = ['myapp.routers.ReadReplicaRouter']

# routers.py
class ReadReplicaRouter:
    def db_for_read(self, model, **hints):
        return 'replica'

    def db_for_write(self, model, **hints):
        return 'default'

    def allow_relation(self, obj1, obj2, **hints):
        return True

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        return db == 'default'
```

## Обработка лага репликации

### Read-After-Write консистентность

```python
class ReadAfterWriteHandler:
    def __init__(self, primary, replicas):
        self.primary = primary
        self.replicas = replicas
        self.write_timestamps = {}  # user_id -> timestamp

    def write(self, user_id, query):
        result = self.primary.execute(query)
        self.write_timestamps[user_id] = time.time()
        return result

    def read(self, user_id, query):
        # Если недавняя запись, читать с primary
        if user_id in self.write_timestamps:
            if time.time() - self.write_timestamps[user_id] < 5:  # 5 сек окно
                return self.primary.execute(query)
            del self.write_timestamps[user_id]

        # Иначе читать с реплики
        return self.get_replica().execute(query)
```

### Session Stickiness

```python
class SessionStickyRouter:
    def __init__(self, primary, replicas, stick_duration=10):
        self.primary = primary
        self.replicas = replicas
        self.stick_duration = stick_duration
        self.sticky_sessions = {}  # session_id -> (primary, expire_time)

    def route_read(self, session_id, query):
        if session_id in self.sticky_sessions:
            db, expire_time = self.sticky_sessions[session_id]
            if time.time() < expire_time:
                return db.execute(query)
            del self.sticky_sessions[session_id]
        return random.choice(self.replicas).execute(query)

    def route_write(self, session_id, query):
        result = self.primary.execute(query)
        # Привязать сессию к primary для чтений
        self.sticky_sessions[session_id] = (
            self.primary,
            time.time() + self.stick_duration
        )
        return result
```

## Мониторинг

```yaml
# Prometheus метрики
db_replica_lag_seconds{replica="replica1"}
db_replica_connections{replica="replica1"}
db_replica_queries_total{replica="replica1"}
db_read_from_primary_total  # Fallback чтения
db_replica_errors_total{replica="replica1"}

# Алертинг
groups:
  - name: read-replicas
    rules:
      - alert: ReplicaLagHigh
        expr: db_replica_lag_seconds > 10
        for: 5m
        labels:
          severity: warning

      - alert: ReplicaDown
        expr: up{job="postgres-replica"} == 0
        for: 1m
        labels:
          severity: critical

      - alert: AllReplicasDown
        expr: count(up{job="postgres-replica"} == 1) == 0
        for: 30s
        labels:
          severity: critical
```

## Best practices

**Делать:**
- Непрерывно мониторить лаг репликации
- Использовать lag-aware маршрутизацию для критичных чтений
- Реализовать read-after-write консистентность для user-facing операций
- Балансировать нагрузку на основе емкости реплик
- Иметь fallback на primary когда все реплики недоступны
- Использовать connection pooling для всех соединений

**Не делать:**
- Отправлять записи на реплики
- Игнорировать лаг репликации в логике приложения
- Использовать реплики для критичных чтений сразу после записи
- Перегружать реплики тяжелыми аналитическими запросами
- Забывать health checks для доступности реплик

## Переменные окружения

```bash
# Primary
DB_PRIMARY_HOST=primary.db.local
DB_PRIMARY_PORT=5432

# Реплики
DB_REPLICA_HOSTS=replica1.db.local,replica2.db.local,replica3.db.local
DB_REPLICA_PORT=5432

# Балансировка нагрузки
DB_LB_STRATEGY=round_robin  # round_robin|weighted|least_conn|lag_aware
DB_LB_MAX_LAG_SECONDS=5
DB_LB_HEALTH_CHECK_INTERVAL=5

# Read-after-write
DB_RAW_CONSISTENCY_WINDOW_SECONDS=5
```

## Дополнительные ресурсы

- [PostgreSQL Replication](https://www.postgresql.org/docs/current/high-availability.html)
- [PgBouncer](https://www.pgbouncer.org/)
- [HAProxy for PostgreSQL](https://www.haproxy.org/)
- [Rails Multiple Databases](https://guides.rubyonrails.org/active_record_multiple_databases.html)
