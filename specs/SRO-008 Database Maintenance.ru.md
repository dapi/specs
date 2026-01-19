# SRO-008 Database Maintenance (Обслуживание баз данных)

## Определение

Обслуживание баз данных включает рутинные операции, которые поддерживают базы данных здоровыми, производительными и надежными. Это включает операции VACUUM, обновление статистики, обслуживание индексов, управление bloat и запланированные задачи очистки.

## Операции обслуживания PostgreSQL

### VACUUM

```sql
-- Стандартный VACUUM (освобождает место, обновляет visibility map)
VACUUM tablename;

-- VACUUM с обновлением статистики
VACUUM ANALYZE tablename;

-- Full VACUUM (перезаписывает всю таблицу, требует эксклюзивной блокировки)
VACUUM FULL tablename;

-- VACUUM конкретных колонок для статистики
VACUUM ANALYZE tablename (column1, column2);

-- Проверка прогресса VACUUM (PostgreSQL 9.6+)
SELECT * FROM pg_stat_progress_vacuum;
```

### ANALYZE

```sql
-- Обновить статистику для планировщика запросов
ANALYZE tablename;

-- Анализ конкретных колонок
ANALYZE tablename (column1, column2);

-- Анализ всей базы данных
ANALYZE;

-- Проверить когда последний раз анализировали
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
-- Переиндексация одного индекса
REINDEX INDEX idx_users_email;

-- Переиндексация всех индексов таблицы
REINDEX TABLE users;

-- Переиндексация всей базы данных
REINDEX DATABASE mydb;

-- Конкурентная переиндексация (PostgreSQL 12+, без блокировок)
REINDEX INDEX CONCURRENTLY idx_users_email;
REINDEX TABLE CONCURRENTLY users;

-- Проверка bloat индексов перед переиндексацией
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS index_scans
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;
```

## Настройка Autovacuum

### Настройки postgresql.conf

```ini
# Включить autovacuum
autovacuum = on

# Рабочие процессы
autovacuum_max_workers = 3

# Как часто запускать (секунды)
autovacuum_naptime = 60

# Пороги для VACUUM
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.2

# Пороги для ANALYZE
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.1

# Cost-based vacuum задержка (throttling)
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 200

# Настройки freeze age
vacuum_freeze_min_age = 50000000
vacuum_freeze_table_age = 150000000
autovacuum_freeze_max_age = 200000000
```

### Настройки для конкретных таблиц

```sql
-- Таблица с высокой нагрузкой на запись: более агрессивный autovacuum
ALTER TABLE events SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_vacuum_threshold = 1000,
    autovacuum_analyze_scale_factor = 0.02,
    autovacuum_analyze_threshold = 500
);

-- Большая таблица: увеличить порог freeze
ALTER TABLE audit_logs SET (
    autovacuum_freeze_max_age = 500000000,
    autovacuum_freeze_table_age = 400000000
);

-- Отключить autovacuum для конкретной таблицы (использовать с осторожностью)
ALTER TABLE temp_data SET (autovacuum_enabled = false);
```

## Управление Bloat

### Обнаружение bloat таблиц

```sql
-- Оценка bloat таблицы
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

-- Более точная оценка bloat с использованием pgstattuple
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

### Обнаружение bloat индексов

```sql
-- Оценка bloat индексов
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

-- Использование pgstattuple для bloat индексов
SELECT * FROM pgstatindex('idx_users_email');
```

### Расширение pg_repack

```sql
-- Установить pg_repack
CREATE EXTENSION pg_repack;

-- Перепаковка таблицы (без блокировок, онлайн операция)
-- Запуск из командной строки:
-- pg_repack -d mydb -t tablename

-- Перепаковка с конкретным индексом
-- pg_repack -d mydb -t tablename -i idx_name

-- Перепаковка всей базы данных
-- pg_repack -d mydb
```

## Окна обслуживания

### Скрипт планового обслуживания

```bash
#!/bin/bash
# maintenance.sh - Скрипт обслуживания базы данных

DB_NAME="mydb"
LOG_FILE="/var/log/db_maintenance.log"
LOCK_FILE="/var/run/db_maintenance.lock"

# Предотвратить параллельный запуск
exec 200>$LOCK_FILE
flock -n 200 || { echo "Обслуживание уже выполняется"; exit 1; }

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> $LOG_FILE
}

log "Начало обслуживания $DB_NAME"

# 1. VACUUM ANALYZE для таблиц с высокой нагрузкой
log "Запуск VACUUM ANALYZE на таблицах с высокой нагрузкой"
psql -d $DB_NAME -c "VACUUM ANALYZE events, sessions, audit_logs;"

# 2. Обновить статистику всех таблиц
log "Запуск ANALYZE на базе данных"
psql -d $DB_NAME -c "ANALYZE;"

# 3. Переиндексация bloated индексов (если bloat > 30%)
log "Проверка и переиндексация bloated индексов"
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
        RAISE NOTICE 'Переиндексирован: %', idx.indexrelname;
    END LOOP;
END\$\$;
EOF

# 4. Очистка старых данных (если применимо)
log "Очистка старых данных"
psql -d $DB_NAME -c "DELETE FROM audit_logs WHERE created_at < NOW() - INTERVAL '90 days';"
psql -d $DB_NAME -c "DELETE FROM sessions WHERE expires_at < NOW() - INTERVAL '7 days';"

# 5. Обновить pg_stat_statements (сбросить если нужно)
log "Проверка pg_stat_statements"
psql -d $DB_NAME -c "SELECT pg_stat_statements_reset();" 2>/dev/null || true

log "Обслуживание завершено для $DB_NAME"
```

### Расписание Cron

```bash
# /etc/cron.d/db_maintenance

# Ежедневное обслуживание в 3 часа ночи
0 3 * * * postgres /opt/scripts/maintenance.sh

# Еженедельный VACUUM FULL на выходных (если нужно)
0 2 * * 0 postgres psql -d mydb -c "VACUUM FULL large_table;"

# Ежемесячная полная переиндексация
0 1 1 * * postgres psql -d mydb -c "REINDEX DATABASE mydb;"
```

## Обслуживание MySQL

### OPTIMIZE TABLE

```sql
-- Дефрагментация таблицы и освобождение места
OPTIMIZE TABLE tablename;

-- Для InnoDB (перестраивает таблицу)
ALTER TABLE tablename ENGINE=InnoDB;

-- Проверка фрагментации таблицы
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
-- Обновить статистику индексов
ANALYZE TABLE tablename;

-- Проверить статистику таблицы
SHOW TABLE STATUS LIKE 'tablename';
```

### CHECK и REPAIR

```sql
-- Проверить таблицу на ошибки
CHECK TABLE tablename;

-- Восстановить таблицу (только MyISAM)
REPAIR TABLE tablename;

-- Для InnoDB используйте innodb_force_recovery или backup/restore
```

## Предотвращение Transaction ID Wraparound

### Мониторинг XID Age

```sql
-- Проверка таблиц, приближающихся к wraparound
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

-- Проверка XID age на уровне базы данных
SELECT
    datname,
    age(datfrozenxid) AS xid_age
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

### Предотвращение Wraparound

```sql
-- Принудительный freeze на критичных таблицах
VACUUM FREEZE tablename;

-- Агрессивные настройки freeze для окна обслуживания
SET vacuum_freeze_min_age = 0;
SET vacuum_freeze_table_age = 0;
VACUUM FREEZE;
```

## Мониторинг обслуживания

### Метрики PostgreSQL

```sql
-- Активность autovacuum
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

-- Текущие процессы VACUUM
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

-- Рабочие процессы autovacuum
SELECT * FROM pg_stat_progress_vacuum;
```

### Правила алертинга

```yaml
# Правила алертинга Prometheus
groups:
  - name: database-maintenance
    rules:
      - alert: HighTableBloat
        expr: pg_stat_user_tables_n_dead_tup > 100000
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Высокое количество dead tuples на {{ $labels.relname }}"

      - alert: AutovacuumNotRunning
        expr: pg_stat_user_tables_last_autovacuum_time > 86400
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Autovacuum не запускался 24ч на {{ $labels.relname }}"

      - alert: XIDWraparoundRisk
        expr: pg_database_xid_age > 500000000
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Риск XID wraparound на {{ $labels.datname }}"

      - alert: LongRunningVacuum
        expr: pg_stat_activity_vacuum_duration_seconds > 3600
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "VACUUM выполняется более 1 часа"
```

## Best practices

**Делать:**
- Правильно настраивать autovacuum для нагрузки
- Регулярно мониторить dead tuples и bloat
- Использовать опции CONCURRENTLY когда возможно
- Планировать обслуживание на периоды низкой нагрузки
- Тестировать процедуры обслуживания на staging
- Мониторить XID age для предотвращения wraparound
- Использовать pg_repack для обслуживания больших таблиц

**Не делать:**
- Отключать autovacuum без веской причины
- Запускать VACUUM FULL в часы пик
- Игнорировать предупреждения о bloat
- Позволять XID age приближаться к порогу wraparound
- Запускать обслуживание без мониторинга влияния
- Забывать обновлять статистику после bulk операций

## Переменные окружения

```bash
# Планирование обслуживания
DB_MAINTENANCE_ENABLED=true
DB_MAINTENANCE_SCHEDULE="0 3 * * *"
DB_MAINTENANCE_LOCK_TIMEOUT=3600

# Настройка autovacuum
DB_AUTOVACUUM_WORKERS=3
DB_AUTOVACUUM_NAPTIME=60
DB_AUTOVACUUM_SCALE_FACTOR=0.2

# Пороги bloat
DB_BLOAT_WARNING_PERCENT=20
DB_BLOAT_CRITICAL_PERCENT=50

# Мониторинг XID
DB_XID_WARNING_AGE=150000000
DB_XID_CRITICAL_AGE=180000000
```

## Дополнительные ресурсы

- [PostgreSQL VACUUM](https://www.postgresql.org/docs/current/sql-vacuum.html)
- [PostgreSQL Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)
- [pg_repack](https://github.com/reorg/pg_repack)
- [MySQL OPTIMIZE TABLE](https://dev.mysql.com/doc/refman/8.0/en/optimize-table.html)
