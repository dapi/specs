# SRG-017 Table Partitioning (Партиционирование таблиц)

## Определение

Партиционирование таблиц разделяет большую таблицу на меньшие, более управляемые части, называемые партициями. Каждая партиция хранит подмножество данных на основе значений ключа партиционирования. В отличие от шардирования, все партиции находятся в одном экземпляре базы данных.

## Когда партиционировать

**Используйте партиционирование когда:**
- Таблица имеет миллионы/миллиарды строк
- Запросы фильтруют по ключу партиции (дата, регион)
- Нужно эффективно архивировать или удалять старые данные
- Хотите улучшить производительность запросов на больших таблицах
- Нужно параллельное выполнение запросов

**Избегайте партиционирования когда:**
- Таблица маленькая (< 1M строк)
- Запросы не фильтруют по ключу партиции
- Сложность перевешивает преимущества

## Стратегии партиционирования

### Range партиционирование

```sql
-- PostgreSQL: Партиционирование по диапазону дат
CREATE TABLE events (
    id bigserial,
    event_type varchar(50),
    payload jsonb,
    created_at timestamptz NOT NULL
) PARTITION BY RANGE (created_at);

-- Создать месячные партиции
CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
CREATE TABLE events_2024_03 PARTITION OF events
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- Default партиция для неожиданных значений
CREATE TABLE events_default PARTITION OF events DEFAULT;
```

### List партиционирование

```sql
-- PostgreSQL: Партиционирование по региону
CREATE TABLE orders (
    id bigserial,
    customer_id int,
    region varchar(20) NOT NULL,
    total decimal(10,2),
    created_at timestamptz
) PARTITION BY LIST (region);

CREATE TABLE orders_us PARTITION OF orders FOR VALUES IN ('US', 'CA');
CREATE TABLE orders_eu PARTITION OF orders FOR VALUES IN ('UK', 'DE', 'FR');
CREATE TABLE orders_asia PARTITION OF orders FOR VALUES IN ('JP', 'CN', 'KR');
CREATE TABLE orders_other PARTITION OF orders DEFAULT;
```

### Hash партиционирование

```sql
-- PostgreSQL: Партиционирование по хэшу для равномерного распределения
CREATE TABLE sessions (
    id uuid PRIMARY KEY,
    user_id int NOT NULL,
    data jsonb,
    expires_at timestamptz
) PARTITION BY HASH (user_id);

CREATE TABLE sessions_0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE sessions_2 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE sessions_3 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

### Составное партиционирование

```sql
-- PostgreSQL: Range + List (суб-партиционирование)
CREATE TABLE logs (
    id bigserial,
    log_level varchar(10) NOT NULL,
    message text,
    created_at timestamptz NOT NULL
) PARTITION BY RANGE (created_at);

-- Месячная партиция
CREATE TABLE logs_2024_01 PARTITION OF logs
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01')
    PARTITION BY LIST (log_level);

-- Суб-партиции по уровню логирования
CREATE TABLE logs_2024_01_error PARTITION OF logs_2024_01
    FOR VALUES IN ('ERROR', 'FATAL');
CREATE TABLE logs_2024_01_info PARTITION OF logs_2024_01
    FOR VALUES IN ('INFO', 'DEBUG', 'WARN');
```

## MySQL партиционирование

```sql
-- Range партиционирование по дате
CREATE TABLE events (
    id BIGINT AUTO_INCREMENT,
    event_type VARCHAR(50),
    payload JSON,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (YEAR(created_at) * 100 + MONTH(created_at)) (
    PARTITION p202401 VALUES LESS THAN (202402),
    PARTITION p202402 VALUES LESS THAN (202403),
    PARTITION p202403 VALUES LESS THAN (202404),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);

-- List партиционирование
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT,
    region VARCHAR(20) NOT NULL,
    total DECIMAL(10,2),
    PRIMARY KEY (id, region)
) PARTITION BY LIST COLUMNS (region) (
    PARTITION p_us VALUES IN ('US', 'CA'),
    PARTITION p_eu VALUES IN ('UK', 'DE', 'FR'),
    PARTITION p_asia VALUES IN ('JP', 'CN', 'KR')
);
```

## Partition Pruning

```sql
-- Запрос, использующий partition pruning
SELECT * FROM events
WHERE created_at >= '2024-02-01' AND created_at < '2024-03-01';

-- EXPLAIN показывает что сканируется только events_2024_02
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM events WHERE created_at = '2024-02-15';

/*
Append
  ->  Seq Scan on events_2024_02
        Filter: (created_at = '2024-02-15')
*/
```

## Автоматизация управления партициями

### PostgreSQL: расширение pg_partman

```sql
-- Установить pg_partman
CREATE EXTENSION pg_partman;

-- Создать партиционированную таблицу
CREATE TABLE metrics (
    id bigserial,
    metric_name varchar(100),
    value numeric,
    recorded_at timestamptz NOT NULL
) PARTITION BY RANGE (recorded_at);

-- Настроить автоматическое управление партициями
SELECT partman.create_parent(
    p_parent_table := 'public.metrics',
    p_control := 'recorded_at',
    p_type := 'native',
    p_interval := 'daily',
    p_premake := 7  -- Создать на 7 дней вперед
);

-- Запускать обслуживание (через cron или pg_cron)
SELECT partman.run_maintenance();
```

### Скрипт автоматического создания партиций

```sql
-- PostgreSQL функция для автоматических месячных партиций
CREATE OR REPLACE FUNCTION create_monthly_partition()
RETURNS void AS $$
DECLARE
    partition_date date;
    partition_name text;
    start_date date;
    end_date date;
BEGIN
    -- Создать партиции на следующие 3 месяца
    FOR i IN 0..2 LOOP
        partition_date := date_trunc('month', CURRENT_DATE + (i || ' month')::interval);
        partition_name := 'events_' || to_char(partition_date, 'YYYY_MM');
        start_date := partition_date;
        end_date := partition_date + interval '1 month';

        -- Проверить существует ли партиция
        IF NOT EXISTS (
            SELECT 1 FROM pg_class WHERE relname = partition_name
        ) THEN
            EXECUTE format(
                'CREATE TABLE %I PARTITION OF events FOR VALUES FROM (%L) TO (%L)',
                partition_name, start_date, end_date
            );
            RAISE NOTICE 'Создана партиция: %', partition_name;
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Запланировать с pg_cron
SELECT cron.schedule('create_partitions', '0 0 1 * *', 'SELECT create_monthly_partition()');
```

## Обслуживание партиций

### Удаление старых партиций

```sql
-- Удалить партицию (быстро, без блокировок на других партициях)
DROP TABLE events_2023_01;

-- Сначала отключить если нужно запросить перед удалением
ALTER TABLE events DETACH PARTITION events_2023_01;
-- ... backup или archive ...
DROP TABLE events_2023_01;
```

### Архивирование партиций

```sql
-- Отключить партицию для архивации
ALTER TABLE events DETACH PARTITION events_2023_12;

-- Переместить в archive tablespace
ALTER TABLE events_2023_12 SET TABLESPACE archive_tablespace;

-- Или экспортировать в файл
COPY events_2023_12 TO '/archive/events_2023_12.csv' WITH CSV HEADER;

-- Затем удалить
DROP TABLE events_2023_12;
```

### Подключение партиций

```sql
-- Подключить партицию (с валидацией ограничений)
ALTER TABLE events ATTACH PARTITION events_2023_12
    FOR VALUES FROM ('2023-12-01') TO ('2024-01-01');
```

## Индексирование партиционированных таблиц

```sql
-- Индекс на партиционированной таблице (создает индекс на каждой партиции)
CREATE INDEX idx_events_type ON events (event_type);

-- Уникальные ограничения должны включать ключ партиции
CREATE UNIQUE INDEX idx_events_id ON events (id, created_at);

-- Локальные индексы (per partition)
CREATE INDEX idx_events_2024_01_payload ON events_2024_01 USING GIN (payload);
```

## Оптимизация запросов

### Запросы, получающие выгоду от партиционирования

```sql
-- Хорошо: Фильтрует по ключу партиции
SELECT * FROM events WHERE created_at >= '2024-02-01' AND created_at < '2024-03-01';

-- Хорошо: Агрегации по диапазону ключа партиции
SELECT date_trunc('day', created_at) AS day, count(*)
FROM events
WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01'
GROUP BY 1;

-- Хорошо: Удаление старых данных (через drop партиции)
ALTER TABLE events DETACH PARTITION events_2023_01;
DROP TABLE events_2023_01;
```

### Запросы без выгоды

```sql
-- Плохо: Нет ключа партиции в WHERE (сканирует все партиции)
SELECT * FROM events WHERE event_type = 'login';

-- Исправление: Добавить ключ партиции в запрос
SELECT * FROM events
WHERE event_type = 'login'
  AND created_at >= '2024-01-01' AND created_at < '2024-02-01';
```

## Мониторинг

```sql
-- Список партиций и размеров
SELECT
    parent.relname AS parent_table,
    child.relname AS partition_name,
    pg_size_pretty(pg_relation_size(child.oid)) AS size,
    pg_stat_user_tables.n_live_tup AS row_count
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
LEFT JOIN pg_stat_user_tables ON child.relname = pg_stat_user_tables.relname
WHERE parent.relname = 'events'
ORDER BY child.relname;

-- Проверить partition pruning в плане запроса
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM events WHERE created_at = '2024-02-15';
```

## Best practices

**Делать:**
- Выбирать ключ партиции на основе паттернов запросов
- Создавать партиции заранее (premake)
- Включать ключ партиции во все индексы
- Использовать partition pruning фильтруя по ключу партиции
- Автоматизировать создание и очистку партиций
- Мониторить размеры партиций для баланса
- Тестировать операции с партициями на staging

**Не делать:**
- Партиционировать маленькие таблицы
- Использовать слишком много партиций (>1000 влияет на planning)
- Забывать создавать default партицию
- Делать запросы без фильтра по ключу партиции
- Создавать партиции вручную без автоматизации
- Игнорировать обслуживание партиций

## Переменные окружения

```bash
# Конфигурация партиционирования
DB_PARTITION_STRATEGY=range
DB_PARTITION_KEY=created_at
DB_PARTITION_INTERVAL=monthly
DB_PARTITION_PREMAKE=3
DB_PARTITION_RETENTION_MONTHS=12

# Обслуживание
DB_PARTITION_MAINTENANCE_ENABLED=true
DB_PARTITION_MAINTENANCE_SCHEDULE="0 2 * * *"
```

## Дополнительные ресурсы

- [PostgreSQL Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [pg_partman Extension](https://github.com/pgpartman/pg_partman)
- [MySQL Partitioning](https://dev.mysql.com/doc/refman/8.0/en/partitioning.html)
- [Partition Pruning](https://www.postgresql.org/docs/current/ddl-partitioning.html#DDL-PARTITION-PRUNING)
