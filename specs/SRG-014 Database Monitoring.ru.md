# SRG-014 Database Monitoring & Metrics (Мониторинг баз данных)

## Определение

Мониторинг базы данных - это непрерывный процесс отслеживания здоровья базы данных, производительности и использования ресурсов. Он позволяет проактивно выявлять проблемы до того, как они повлияют на приложения.

## Категории ключевых метрик

### Метрики соединений

```
# PostgreSQL
db_connections_active (gauge)          # Активные соединения
db_connections_idle (gauge)            # Простаивающие соединения
db_connections_idle_in_transaction (gauge) # Idle in transaction
db_connections_max (gauge)             # Настройка max_connections
db_connections_used_percent (gauge)    # Процент использования

# Пороги
WARNING: connections_used > 70%
CRITICAL: connections_used > 85%
```

```sql
-- PostgreSQL: Текущие соединения
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;

-- Возраст соединений
SELECT pid, now() - backend_start AS connection_age,
       state, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY connection_age DESC;
```

### Метрики производительности запросов

```
db_query_duration_seconds (histogram)  # Время выполнения запросов
db_queries_total (counter)             # Всего выполненных запросов
db_slow_queries_total (counter)        # Запросы > порога
db_rows_returned (counter)             # Возвращено строк
db_rows_affected (counter)             # Вставлено/обновлено/удалено строк

# Пороги
SLOW_QUERY_THRESHOLD: 1 секунда
WARNING: avg_query_time > 100ms
CRITICAL: avg_query_time > 500ms
```

### Метрики транзакций

```
db_transactions_committed (counter)    # Закоммиченные транзакции
db_transactions_rolled_back (counter)  # Откаченные транзакции
db_transaction_duration_seconds (histogram) # Длительность транзакций
db_deadlocks_total (counter)           # Количество deadlock'ов
db_lock_wait_seconds (histogram)       # Время ожидания блокировки

# Пороги
WARNING: rollback_rate > 1%
CRITICAL: deadlocks > 0 в минуту
```

### Метрики хранилища

```
db_size_bytes (gauge)                  # Размер базы данных
db_table_size_bytes (gauge)            # Размер таблицы
db_index_size_bytes (gauge)            # Размер индекса
db_bloat_bytes (gauge)                 # Bloat таблиц/индексов
db_disk_usage_percent (gauge)          # Использование диска

# Пороги
WARNING: disk_usage > 70%
CRITICAL: disk_usage > 85%
WARNING: bloat > 20%
```

### Метрики кэша

```
db_cache_hit_ratio (gauge)             # Buffer cache hit ratio
db_index_hit_ratio (gauge)             # Index hit ratio
db_shared_buffers_used (gauge)         # Использование shared buffers

# Пороги
WARNING: cache_hit_ratio < 95%
CRITICAL: cache_hit_ratio < 90%
```

## Мониторинг PostgreSQL

### Расширение pg_stat_statements

```sql
-- Включить расширение
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Конфигурация (postgresql.conf)
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
pg_stat_statements.max = 10000

-- Топ 10 самых медленных запросов
SELECT query,
       calls,
       total_exec_time / 1000 AS total_seconds,
       mean_exec_time / 1000 AS avg_seconds,
       rows,
       100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Самые частые запросы
SELECT query, calls, rows, mean_exec_time
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- Запросы, потребляющие больше всего времени
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
-- Статистика таблиц
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

-- Таблицы, нуждающиеся в индексе (высокий seq_scan, низкий idx_scan)
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
-- Неиспользуемые индексы
SELECT schemaname, relname, indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS idx_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Статистика использования индексов
SELECT relname AS table_name,
       indexrelname AS index_name,
       idx_scan,
       idx_tup_read,
       idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

### Мониторинг блокировок

```sql
-- Текущие блокировки
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

-- Долго выполняющиеся запросы
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes'
  AND state != 'idle';
```

## Мониторинг MySQL

### Performance Schema

```sql
-- Включить Performance Schema (my.cnf)
performance_schema = ON

-- Топ запросов по общему времени
SELECT DIGEST_TEXT,
       COUNT_STAR AS calls,
       SUM_TIMER_WAIT / 1000000000000 AS total_seconds,
       AVG_TIMER_WAIT / 1000000000000 AS avg_seconds,
       SUM_ROWS_EXAMINED,
       SUM_ROWS_SENT
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- Статистика I/O таблиц
SELECT OBJECT_SCHEMA, OBJECT_NAME,
       COUNT_READ, COUNT_WRITE,
       SUM_TIMER_READ / 1000000000000 AS read_seconds,
       SUM_TIMER_WRITE / 1000000000000 AS write_seconds
FROM performance_schema.table_io_waits_summary_by_table
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

### Метрики InnoDB

```sql
-- InnoDB buffer pool hit ratio
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';

-- Расчет hit ratio
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

### Основные панели

```
1. Статус пула соединений
   - Active/Idle/Max соединения
   - Процент использования соединений

2. Производительность запросов
   - Запросов в секунду
   - Средняя задержка запросов
   - Количество медленных запросов

3. Метрики транзакций
   - Транзакций в секунду
   - Соотношение Commit/Rollback
   - Количество deadlock'ов

4. Хранилище
   - Размер базы данных
   - Размеры таблиц (топ 10)
   - Использование диска

5. Производительность кэша
   - Buffer cache hit ratio
   - Index hit ratio

6. Репликация (если применимо)
   - Лаг репликации
   - Статус реплик
```

### Пример Dashboard JSON

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

## Правила алертинга

```yaml
groups:
  - name: database
    rules:
      # Алерты соединений
      - alert: DatabaseConnectionsHigh
        expr: pg_stat_activity_count / pg_settings_max_connections > 0.7
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Соединения базы данных выше 70%"

      - alert: DatabaseConnectionsCritical
        expr: pg_stat_activity_count / pg_settings_max_connections > 0.85
        for: 2m
        labels:
          severity: critical

      # Алерты производительности
      - alert: SlowQueries
        expr: rate(pg_stat_statements_seconds_total[5m]) > 1
        for: 10m
        labels:
          severity: warning

      - alert: DatabaseDeadlock
        expr: increase(pg_stat_database_deadlocks[5m]) > 0
        labels:
          severity: critical

      # Алерты хранилища
      - alert: DatabaseDiskUsageHigh
        expr: pg_database_size_bytes / node_filesystem_size_bytes > 0.7
        for: 10m
        labels:
          severity: warning

      # Алерты кэша
      - alert: LowCacheHitRatio
        expr: pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read) < 0.95
        for: 15m
        labels:
          severity: warning

      # Алерты репликации
      - alert: ReplicationLagHigh
        expr: pg_replication_lag > 30
        for: 5m
        labels:
          severity: warning
```

## Инструменты мониторинга

| Инструмент | Описание | Случай использования |
|------------|----------|---------------------|
| pg_stat_statements | Статистика запросов | Оптимизация запросов |
| pgBadger | Анализатор логов | Исторический анализ |
| pg_activity | Живой мониторинг | Отладка в реальном времени |
| Percona PMM | Полный мониторинг | Enterprise мониторинг |
| DataDog | Cloud мониторинг | SaaS решение |
| New Relic | APM + DB | Корреляция с приложением |

## Best practices

**Делать:**
- Включать pg_stat_statements во всех окружениях
- Настраивать алертинг для критичных метрик
- Создавать дашборды для ключевых метрик
- Хранить историю метрик для планирования емкости
- Коррелировать метрики БД с метриками приложения
- Мониторить как агрегированные метрики, так и per-query
- Отслеживать тренды медленных запросов

**Не делать:**
- Игнорировать деградацию cache hit ratio
- Пропускать мониторинг пула соединений
- Забывать мониторить дисковое пространство
- Слишком много алертов на временные всплески
- Игнорировать статистику использования индексов
- Пропускать мониторинг блокировок

## Переменные окружения

```bash
# Конфигурация мониторинга
DB_SLOW_QUERY_THRESHOLD_MS=1000
DB_MONITORING_INTERVAL_SECONDS=15
DB_STATS_RETENTION_DAYS=30

# Пороги алертинга
DB_ALERT_CONNECTIONS_WARNING=70
DB_ALERT_CONNECTIONS_CRITICAL=85
DB_ALERT_CACHE_HIT_WARNING=95
DB_ALERT_DISK_USAGE_WARNING=70
DB_ALERT_REPLICATION_LAG_WARNING=30
```

## Дополнительные ресурсы

- [PostgreSQL Statistics Collector](https://www.postgresql.org/docs/current/monitoring-stats.html)
- [MySQL Performance Schema](https://dev.mysql.com/doc/refman/8.0/en/performance-schema.html)
- [postgres_exporter](https://github.com/prometheus-community/postgres_exporter)
- [pgBadger](https://github.com/darold/pgbadger)
- [Percona Monitoring and Management](https://www.percona.com/software/database-tools/percona-monitoring-and-management)
