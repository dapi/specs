# SRG-015 Query Optimization (Оптимизация запросов)

## Определение

Оптимизация запросов - это процесс улучшения производительности запросов к базе данных путем анализа планов выполнения, создания подходящих индексов и переписывания запросов для уменьшения потребления ресурсов и времени выполнения.

## EXPLAIN ANALYZE

### PostgreSQL

```sql
-- Базовый explain
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';

-- Со статистикой выполнения
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

-- Полный анализ с буферами
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM users WHERE email = 'test@example.com';

-- JSON формат для инструментов
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM users WHERE email = 'test@example.com';
```

### Понимание вывода EXPLAIN

```sql
-- Пример вывода
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;

/*
Seq Scan on orders  (cost=0.00..1234.00 rows=100 width=200) (actual time=0.015..12.500 rows=95 loops=1)
  Filter: (user_id = 123)
  Rows Removed by Filter: 9905
Planning Time: 0.100 ms
Execution Time: 12.600 ms
*/
```

**Ключевые метрики:**
- `cost=0.00..1234.00` - оценочная стоимость запуска и общая
- `rows=100` - оценочное количество возвращаемых строк
- `actual time=0.015..12.500` - фактическое время запуска и общее (мс)
- `rows=95` - фактическое количество возвращенных строк
- `loops=1` - количество итераций цикла

### MySQL

```sql
-- Базовый explain
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';

-- Расширенный explain
EXPLAIN EXTENDED SELECT * FROM users WHERE email = 'test@example.com';

-- JSON формат
EXPLAIN FORMAT=JSON SELECT * FROM users WHERE email = 'test@example.com';

-- Анализ выполнения
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```

## Типы сканирования

### Sequential Scan (Seq Scan)

```sql
-- Плохо: Полное сканирование таблицы
Seq Scan on users  (cost=0.00..10000.00 rows=100000 width=100)

-- Когда происходит:
-- 1. Нет подходящего индекса
-- 2. Запрос возвращает большую часть таблицы (>10-20%)
-- 3. Маленькая таблица, где seq scan быстрее
```

### Index Scan

```sql
-- Хорошо: Использование индекса
Index Scan using users_email_idx on users  (cost=0.42..8.44 rows=1 width=100)
  Index Cond: (email = 'test@example.com'::text)

-- Читает индекс, затем извлекает строки из таблицы
```

### Index Only Scan

```sql
-- Лучше всего: Все данные из индекса, без обращения к таблице
Index Only Scan using users_email_name_idx on users  (cost=0.42..4.44 rows=1 width=50)
  Index Cond: (email = 'test@example.com'::text)

-- Требует: все выбираемые колонки в индексе + недавний VACUUM
```

### Bitmap Scan

```sql
-- Хорошо для множественных условий или range запросов
Bitmap Heap Scan on orders  (cost=50.00..500.00 rows=1000 width=100)
  Recheck Cond: (status = 'pending')
  ->  Bitmap Index Scan on orders_status_idx  (cost=0.00..49.75 rows=1000 width=0)
        Index Cond: (status = 'pending')

-- Двухфазный: построить bitmap, затем извлечь строки
```

## Типы индексов

### B-Tree Index (По умолчанию)

```sql
-- Лучший для: equality и range запросов
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_date ON orders(created_at);

-- Поддерживает: =, <, >, <=, >=, BETWEEN, IN, IS NULL
-- Также: LIKE 'prefix%' (но не '%suffix')
```

### Составной индекс

```sql
-- Многоколоночный индекс
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Порядок важен! Индекс может использоваться для:
-- WHERE user_id = 123
-- WHERE user_id = 123 AND status = 'pending'
-- Не может эффективно использоваться для:
-- WHERE status = 'pending' (без user_id)
```

### Частичный индекс

```sql
-- Индекс подмножества строк
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';

-- Меньший индекс, быстрее для специфичных запросов
-- Полезен только при фильтрации по условию
```

### Expression Index

```sql
-- Индекс на результат выражения
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Запрос должен использовать то же выражение
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';
```

### GIN Index (PostgreSQL)

```sql
-- Для массивов, JSONB, полнотекстового поиска
CREATE INDEX idx_posts_tags ON posts USING GIN(tags);
CREATE INDEX idx_data_jsonb ON data USING GIN(metadata jsonb_path_ops);

-- Полнотекстовый поиск
CREATE INDEX idx_articles_search ON articles USING GIN(to_tsvector('english', title || ' ' || body));
```

### Hash Index

```sql
-- Только для сравнений на равенство
CREATE INDEX idx_sessions_token ON sessions USING HASH(token);

-- Быстрее B-tree для equality, но:
-- - Нет range запросов
-- - Не логировался в WAL до PostgreSQL 10
```

## Распространенные проблемы производительности

### Проблема N+1

```sql
-- Плохо: N+1 запросов
-- 1 запрос для users
SELECT * FROM users LIMIT 10;
-- N запросов для orders (один на пользователя)
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 2;
-- ... повторяется 10 раз

-- Хорошо: JOIN или подзапрос
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
LIMIT 10;

-- Или batch запрос
SELECT * FROM orders WHERE user_id IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
```

### Отсутствующий индекс

```sql
-- Определить отсутствующие индексы (PostgreSQL)
SELECT relname AS table_name,
       seq_scan,
       seq_tup_read,
       idx_scan,
       seq_tup_read / seq_scan AS avg_rows_per_scan
FROM pg_stat_user_tables
WHERE seq_scan > 100
  AND seq_tup_read / seq_scan > 1000
ORDER BY seq_tup_read DESC;
```

### Неиспользуемый индекс

```sql
-- Найти неиспользуемые индексы
SELECT schemaname, relname, indexrelname,
       idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS idx_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Неявное преобразование типов

```sql
-- Плохо: Несоответствие типов предотвращает использование индекса
-- user_id это INTEGER, но сравниваем со STRING
SELECT * FROM orders WHERE user_id = '123';

-- Хорошо: Соответствующие типы
SELECT * FROM orders WHERE user_id = 123;
```

### Функция на индексированной колонке

```sql
-- Плохо: Функция предотвращает использование индекса
SELECT * FROM users WHERE YEAR(created_at) = 2024;

-- Хорошо: Range запрос использует индекс
SELECT * FROM users
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
```

### LIKE с ведущим wildcard

```sql
-- Плохо: Не может использовать индекс
SELECT * FROM users WHERE email LIKE '%@gmail.com';

-- Хорошо: Может использовать индекс
SELECT * FROM users WHERE email LIKE 'john%';

-- Альтернатива: Использовать полнотекстовый поиск или триграмный индекс
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_users_email_trgm ON users USING GIN(email gin_trgm_ops);
SELECT * FROM users WHERE email LIKE '%@gmail.com';  -- Теперь использует индекс
```

## Техники переписывания запросов

### Использовать EXISTS вместо IN

```sql
-- Медленнее: IN с подзапросом
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 1000);

-- Быстрее: EXISTS
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.total > 1000);
```

### Избегать SELECT *

```sql
-- Плохо: Получает все колонки
SELECT * FROM users WHERE id = 123;

-- Хорошо: Получить только нужные колонки (включает index-only scan)
SELECT id, name, email FROM users WHERE id = 123;
```

### Ограничивать результаты раньше

```sql
-- Плохо: Сортировать все, затем ограничить
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;

-- Лучше с индексом: Индекс обеспечивает порядок
CREATE INDEX idx_orders_created ON orders(created_at DESC);
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;  -- Index scan, без сортировки
```

### Использовать UNION ALL вместо UNION

```sql
-- Медленнее: UNION удаляет дубликаты (сортирует)
SELECT email FROM users UNION SELECT email FROM contacts;

-- Быстрее: UNION ALL сохраняет дубликаты (без сортировки)
SELECT email FROM users UNION ALL SELECT email FROM contacts;
```

### Оптимизировать JOIN'ы

```sql
-- Обеспечить индексы на join колонках
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Поставить меньшую таблицу первой (хотя оптимизатор обычно справляется)
SELECT o.* FROM orders o
JOIN users u ON u.id = o.user_id
WHERE u.status = 'active';

-- Для больших join'ов рассмотреть batch обработку
```

## Анализ медленных запросов

### PostgreSQL: pg_stat_statements

```sql
-- Топ 10 по общему времени
SELECT query,
       calls,
       total_exec_time / 1000 AS total_sec,
       mean_exec_time / 1000 AS avg_sec,
       rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Сбросить статистику
SELECT pg_stat_statements_reset();
```

### Slow Query Log

```sql
-- PostgreSQL (postgresql.conf)
log_min_duration_statement = 1000  -- Логировать запросы > 1 секунды
log_statement = 'none'              -- Не логировать все запросы

-- MySQL (my.cnf)
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
```

## Обслуживание индексов

### Перестроить раздутые индексы

```sql
-- Проверить bloat индекса (PostgreSQL)
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Перестроить индекс (блокирующий)
REINDEX INDEX idx_name;

-- Перестроить конкурентно (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_name;
```

### Обновить статистику

```sql
-- PostgreSQL
ANALYZE users;
ANALYZE;  -- Все таблицы

-- MySQL
ANALYZE TABLE users;
```

## Best practices

**Делать:**
- Всегда использовать EXPLAIN ANALYZE для медленных запросов
- Создавать индексы для часто фильтруемых колонок
- Использовать составные индексы для многоколоночных фильтров
- Поддерживать статистику актуальной (ANALYZE)
- Мониторить тренды производительности запросов
- Использовать частичные индексы для фильтрованных запросов
- Рассмотреть covering индексы для index-only scan
- Тестировать изменения индексов на staging

**Не делать:**
- Создавать слишком много индексов (замедляет запись)
- Использовать функции на индексированных колонках в WHERE
- Игнорировать неявные преобразования типов
- SELECT * когда нужны конкретные колонки
- Использовать LIKE '%pattern%' без триграмного индекса
- Забывать удалять неиспользуемые индексы
- Пропускать тестирование с production-подобными объемами данных

## Целевые показатели производительности

```
# Целевое время ответа
Быстрые запросы: < 10ms
Обычные запросы: < 100ms
Сложные запросы: < 1 секунды
Отчеты/Аналитика: < 10 секунд

# Использование индексов
Cache hit ratio: > 99%
Index hit ratio: > 95%
Seq scan на больших таблицах: < 5% запросов
```

## Дополнительные ресурсы

- [PostgreSQL EXPLAIN Documentation](https://www.postgresql.org/docs/current/using-explain.html)
- [Use The Index, Luke](https://use-the-index-luke.com/)
- [MySQL Query Optimization](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
- [pgMustard - EXPLAIN Analyzer](https://www.pgmustard.com/)
- [explain.depesz.com](https://explain.depesz.com/)
