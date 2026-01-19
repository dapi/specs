# SRG-019 Database Performance Tuning (Настройка производительности баз данных)

## Определение

Настройка производительности базы данных включает конфигурирование параметров базы данных, распределение памяти, настройки хранения и системных ресурсов для оптимизации производительности под конкретные рабочие нагрузки. Это включает стратегии настройки как PostgreSQL, так и MySQL.

## Конфигурация памяти PostgreSQL

### Shared Buffers

```ini
# postgresql.conf
# Разделяемая память для кэширования страниц данных
# Рекомендуется: 25% от RAM системы, до 8GB для большинства нагрузок

# Для системы с 32GB RAM
shared_buffers = 8GB

# Для системы с 16GB RAM
shared_buffers = 4GB

# Для системы с 8GB RAM
shared_buffers = 2GB
```

### Work Memory

```ini
# postgresql.conf
# Память для операций сортировки, хэш-таблиц на запрос
# Осторожно: это на операцию, не на соединение

# По умолчанию для OLTP нагрузок
work_mem = 64MB

# Для сложных запросов с множеством сортировок/join'ов
work_mem = 256MB

# Расчет максимального использования памяти:
# max_connections * work_mem * operations_per_query
```

### Maintenance Work Memory

```ini
# postgresql.conf
# Память для VACUUM, CREATE INDEX, ALTER TABLE

# Для систем с большим количеством операций обслуживания
maintenance_work_mem = 2GB

# Для небольших систем
maintenance_work_mem = 512MB

# Рабочие процессы autovacuum делят эту память
autovacuum_work_mem = 512MB
```

### Effective Cache Size

```ini
# postgresql.conf
# Подсказка планировщику запросов о кэше ОС
# Установите ~75% от общего объема RAM

# Для 32GB RAM
effective_cache_size = 24GB

# Для 16GB RAM
effective_cache_size = 12GB
```

## Планировщик запросов PostgreSQL

### Параметры стоимости

```ini
# postgresql.conf
# Настройте в зависимости от типа хранилища

# Для SSD хранилища
random_page_cost = 1.1
seq_page_cost = 1.0

# Для HDD хранилища (по умолчанию)
random_page_cost = 4.0
seq_page_cost = 1.0

# Стоимость CPU (обычно оставить по умолчанию)
cpu_tuple_cost = 0.01
cpu_index_tuple_cost = 0.005
cpu_operator_cost = 0.0025

# Пороги параллельных запросов
parallel_tuple_cost = 0.1
parallel_setup_cost = 1000.0
```

### Настройки параллельных запросов

```ini
# postgresql.conf
# Включить параллельные запросы для больших таблиц

# Максимум параллельных рабочих процессов на запрос
max_parallel_workers_per_gather = 4

# Всего параллельных рабочих процессов
max_parallel_workers = 8

# Минимальный размер таблицы для параллельного сканирования
min_parallel_table_scan_size = 8MB

# Минимальный размер индекса для параллельного сканирования
min_parallel_index_scan_size = 512kB
```

## Конфигурация WAL PostgreSQL

### Write-Ahead Logging

```ini
# postgresql.conf

# Уровень WAL (для репликации установите 'replica' или 'logical')
wal_level = replica

# Буферы WAL (по умолчанию обычно достаточно)
wal_buffers = 64MB

# Настройки checkpoint
checkpoint_completion_target = 0.9
checkpoint_timeout = 10min
max_wal_size = 4GB
min_wal_size = 1GB

# Сжатие WAL (PostgreSQL 15+)
wal_compression = zstd

# Синхронный коммит (компромисс между надежностью и скоростью)
synchronous_commit = on  # самый безопасный
# synchronous_commit = off  # самый быстрый, но риск потери данных
```

### Задержка коммита

```ini
# postgresql.conf
# Группировка коммитов для высокопроизводительного OLTP

commit_delay = 10  # микросекунды
commit_siblings = 5  # мин. конкурентных транзакций
```

## Настройки соединений PostgreSQL

```ini
# postgresql.conf

# Максимум соединений (используйте connection pooler для больших чисел)
max_connections = 200

# Зарезервировано для суперпользователя
superuser_reserved_connections = 3

# Таймаут соединения
tcp_keepalives_idle = 60
tcp_keepalives_interval = 10
tcp_keepalives_count = 6

# Таймаут выполнения (предотвращает зависшие запросы)
statement_timeout = 30000  # 30 секунд

# Таймаут блокировки
lock_timeout = 10000  # 10 секунд

# Таймаут бездействующей транзакции (PostgreSQL 14+)
idle_in_transaction_session_timeout = 60000  # 1 минута
```

## Конфигурация MySQL/InnoDB

### Buffer Pool

```ini
# my.cnf
[mysqld]
# InnoDB buffer pool - основной кэш памяти
# Установите 70-80% доступной RAM для выделенного сервера БД

# Для 32GB RAM
innodb_buffer_pool_size = 24G

# Для 16GB RAM
innodb_buffer_pool_size = 12G

# Количество инстансов buffer pool (для buffer pool >1GB)
innodb_buffer_pool_instances = 8

# Размер чанка buffer pool
innodb_buffer_pool_chunk_size = 128M
```

### Настройки логов InnoDB

```ini
# my.cnf
[mysqld]
# Размер redo log (больше = лучшая производительность записи, медленнее восстановление)
innodb_log_file_size = 2G
innodb_log_files_in_group = 2

# Размер буфера логов
innodb_log_buffer_size = 64M

# Метод flush
innodb_flush_method = O_DIRECT

# Flush log при коммите транзакции
# 1 = самый безопасный (ACID compliant)
# 2 = flush только в кэш ОС
# 0 = flush каждую секунду (самый быстрый, риск потери данных)
innodb_flush_log_at_trx_commit = 1
```

### Настройки I/O InnoDB

```ini
# my.cnf
[mysqld]
# I/O мощность (IOPS которые может обрабатывать ваше хранилище)
# Для SSD
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000

# Для HDD
innodb_io_capacity = 200
innodb_io_capacity_max = 400

# Потоки чтения/записи
innodb_read_io_threads = 4
innodb_write_io_threads = 4

# Потоки purge
innodb_purge_threads = 4
```

### Настройки соединений MySQL

```ini
# my.cnf
[mysqld]
# Максимум соединений
max_connections = 500

# Кэш потоков
thread_cache_size = 50

# Таймаут соединения
wait_timeout = 28800
interactive_timeout = 28800

# Максимальный размер пакета (для больших запросов/blob)
max_allowed_packet = 64M
```

## Настройка на уровне ОС

### Параметры ядра Linux

```bash
# /etc/sysctl.conf

# Виртуальная память
vm.swappiness = 10
vm.dirty_ratio = 40
vm.dirty_background_ratio = 10

# Сеть
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.core.netdev_max_backlog = 65535

# Huge pages для PostgreSQL
vm.nr_hugepages = 4096  # Рассчитайте на основе shared_buffers

# Файловые дескрипторы
fs.file-max = 2097152
```

### Планировщик дискового I/O

```bash
# Для SSD (NVMe)
echo none > /sys/block/nvme0n1/queue/scheduler

# Для SSD (SATA)
echo noop > /sys/block/sda/queue/scheduler

# Проверить текущий планировщик
cat /sys/block/sda/queue/scheduler
```

### Опции монтирования файловой системы

```bash
# /etc/fstab для директории данных PostgreSQL
/dev/sda1 /var/lib/postgresql ext4 noatime,nodiratime,nobarrier 0 2

# Для XFS (рекомендуется для PostgreSQL)
/dev/sda1 /var/lib/postgresql xfs noatime,nodiratime,logbufs=8 0 2
```

## Настройка под конкретные нагрузки

### OLTP нагрузка

```ini
# PostgreSQL для OLTP
shared_buffers = 8GB
work_mem = 64MB
effective_cache_size = 24GB
random_page_cost = 1.1
max_connections = 500
synchronous_commit = on
wal_buffers = 64MB
checkpoint_completion_target = 0.9
```

### OLAP/Analytics нагрузка

```ini
# PostgreSQL для OLAP
shared_buffers = 8GB
work_mem = 2GB  # Выше для сложных запросов
effective_cache_size = 24GB
random_page_cost = 1.1
max_connections = 50  # Меньше соединений
max_parallel_workers_per_gather = 8
parallel_tuple_cost = 0.001
effective_io_concurrency = 200
```

### Смешанная нагрузка

```ini
# PostgreSQL для смешанной нагрузки
shared_buffers = 8GB
work_mem = 256MB
effective_cache_size = 24GB
random_page_cost = 1.1
max_connections = 200
max_parallel_workers_per_gather = 4
```

## Запросы мониторинга производительности

### PostgreSQL

```sql
-- Проверить коэффициент попадания в кэш буферов
SELECT
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit)  as heap_hit,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as hit_ratio
FROM pg_statio_user_tables;

-- Проверить использование индексов
SELECT
    schemaname,
    tablename,
    indexrelname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Проверить статистику таблиц
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

-- Проверить текущие настройки
SELECT name, setting, unit, context
FROM pg_settings
WHERE name IN (
    'shared_buffers', 'work_mem', 'effective_cache_size',
    'random_page_cost', 'max_connections', 'wal_buffers'
);
```

### MySQL

```sql
-- Проверить статус InnoDB buffer pool
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- Проверить коэффициент попадания в buffer pool
SELECT
    (1 - (
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    )) * 100 AS buffer_pool_hit_ratio;

-- Проверить статус InnoDB
SHOW ENGINE INNODB STATUS\G

-- Проверить текущие настройки
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'innodb_log_file_size';
SHOW VARIABLES LIKE 'max_connections';
```

## Инструменты автоматической настройки

### PGTune

```bash
# Генерация конфигурации PostgreSQL
# https://pgtune.leopard.in.ua/

# Пример вывода для 32GB RAM, SSD, OLTP нагрузка:
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
# Установка и запуск MySQLTuner
wget https://raw.githubusercontent.com/major/MySQLTuner-perl/master/mysqltuner.pl
perl mysqltuner.pl --host localhost --user root --pass password

# Просмотрите рекомендации и применяйте осторожно
```

## Best practices

**Делать:**
- Начинать с консервативных настроек и корректировать на основе мониторинга
- Тестировать изменения конфигурации на staging перед production
- Мониторить использование памяти и корректировать соответственно
- Использовать connection pooling для большого количества соединений
- Устанавливать appropriate таймауты для предотвращения зависших запросов
- Настраивать под конкретную нагрузку (OLTP vs OLAP)
- Поддерживать ОС и ПО БД обновленными

**Не делать:**
- Копировать конфигурации с других систем без понимания
- Устанавливать work_mem слишком высоко (умножается на соединения)
- Игнорировать оценки стоимости планировщика запросов
- Отключать synchronous_commit без понимания рисков
- Настраивать параметры без базовых измерений
- Забывать перезапустить БД после изменения конфигурации
- Выделять слишком много памяти (оставьте место для кэша ОС)

## Переменные окружения

```bash
# Настройки памяти PostgreSQL
POSTGRES_SHARED_BUFFERS=8GB
POSTGRES_WORK_MEM=64MB
POSTGRES_MAINTENANCE_WORK_MEM=2GB
POSTGRES_EFFECTIVE_CACHE_SIZE=24GB

# Настройки производительности PostgreSQL
POSTGRES_RANDOM_PAGE_COST=1.1
POSTGRES_MAX_PARALLEL_WORKERS=8
POSTGRES_MAX_CONNECTIONS=200

# Настройки MySQL/InnoDB
MYSQL_INNODB_BUFFER_POOL_SIZE=24G
MYSQL_INNODB_LOG_FILE_SIZE=2G
MYSQL_MAX_CONNECTIONS=500

# Тип нагрузки
DB_WORKLOAD_TYPE=oltp  # oltp|olap|mixed
```

## Дополнительные ресурсы

- [PostgreSQL Performance Tips](https://www.postgresql.org/docs/current/performance-tips.html)
- [PGTune](https://pgtune.leopard.in.ua/)
- [MySQL Performance Tuning](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
- [MySQLTuner](https://github.com/major/MySQLTuner-perl)
- [Linux Performance Tuning for PostgreSQL](https://www.postgresql.org/docs/current/kernel-resources.html)
