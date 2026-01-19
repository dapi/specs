# SRP-009 Materialized Views & Caching (Материализованные представления и кэширование)

## Определение

Материализованные представления - это объекты базы данных, которые физически хранят результат запроса на диске, в отличие от обычных представлений, которые выполняют запрос при каждом обращении. В сочетании со стратегиями кэширования они значительно улучшают производительность чтения для сложных запросов и агрегаций.

## Материализованные представления PostgreSQL

### Создание материализованных представлений

```sql
-- Базовое материализованное представление
CREATE MATERIALIZED VIEW mv_daily_sales AS
SELECT
    date_trunc('day', created_at) AS sale_date,
    product_id,
    SUM(quantity) AS total_quantity,
    SUM(amount) AS total_amount,
    COUNT(*) AS order_count
FROM orders
WHERE created_at >= NOW() - INTERVAL '1 year'
GROUP BY 1, 2;

-- Без данных (создать структуру, заполнить позже)
CREATE MATERIALIZED VIEW mv_monthly_report AS
SELECT
    date_trunc('month', created_at) AS month,
    category_id,
    SUM(revenue) AS total_revenue
FROM sales
GROUP BY 1, 2
WITH NO DATA;

-- Заполнить позже
REFRESH MATERIALIZED VIEW mv_monthly_report;
```

### Индексирование материализованных представлений

```sql
-- Создать индексы на материализованном представлении для быстрого поиска
CREATE UNIQUE INDEX idx_mv_daily_sales_date_product
    ON mv_daily_sales (sale_date, product_id);

CREATE INDEX idx_mv_daily_sales_product
    ON mv_daily_sales (product_id);

CREATE INDEX idx_mv_daily_sales_amount
    ON mv_daily_sales (total_amount DESC);
```

### Обновление материализованных представлений

```sql
-- Полное обновление (блокирует чтение во время обновления)
REFRESH MATERIALIZED VIEW mv_daily_sales;

-- Конкурентное обновление (требует уникальный индекс, без блокировки)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_sales;

-- Проверить когда материализованное представление последний раз обновлялось
SELECT
    schemaname,
    matviewname,
    hasindexes,
    ispopulated,
    definition
FROM pg_matviews
WHERE matviewname = 'mv_daily_sales';
```

### Автоматическое обновление с pg_cron

```sql
-- Установить расширение pg_cron
CREATE EXTENSION pg_cron;

-- Запланировать ежечасное обновление
SELECT cron.schedule(
    'refresh_mv_daily_sales',
    '0 * * * *',  -- Каждый час
    'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_sales'
);

-- Запланировать ежедневное обновление в 2 часа ночи
SELECT cron.schedule(
    'refresh_mv_monthly_report',
    '0 2 * * *',
    'REFRESH MATERIALIZED VIEW mv_monthly_report'
);

-- Список запланированных задач
SELECT * FROM cron.job;

-- Удалить запланированную задачу
SELECT cron.unschedule('refresh_mv_daily_sales');
```

### Обновление на основе триггеров

```sql
-- Функция для обновления материализованного представления
CREATE OR REPLACE FUNCTION refresh_mv_on_change()
RETURNS TRIGGER AS $$
BEGIN
    -- Обновить асинхронно используя pg_notify
    PERFORM pg_notify('refresh_mv', 'mv_daily_sales');
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Триггер на исходной таблице
CREATE TRIGGER trg_refresh_mv_daily_sales
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH STATEMENT
    EXECUTE FUNCTION refresh_mv_on_change();

-- Фоновый рабочий процесс слушает и обновляет
-- (Реализуется в коде приложения)
```

## Материализованные представления MySQL (эмуляция)

MySQL не имеет нативных материализованных представлений, но их можно эмулировать:

```sql
-- Создать сводную таблицу (действует как материализованное представление)
CREATE TABLE mv_daily_sales (
    sale_date DATE NOT NULL,
    product_id INT NOT NULL,
    total_quantity INT NOT NULL,
    total_amount DECIMAL(15,2) NOT NULL,
    order_count INT NOT NULL,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (sale_date, product_id),
    INDEX idx_product (product_id),
    INDEX idx_amount (total_amount DESC)
) ENGINE=InnoDB;

-- Процедура обновления
DELIMITER //
CREATE PROCEDURE refresh_mv_daily_sales()
BEGIN
    TRUNCATE TABLE mv_daily_sales;

    INSERT INTO mv_daily_sales (sale_date, product_id, total_quantity, total_amount, order_count)
    SELECT
        DATE(created_at) AS sale_date,
        product_id,
        SUM(quantity) AS total_quantity,
        SUM(amount) AS total_amount,
        COUNT(*) AS order_count
    FROM orders
    WHERE created_at >= DATE_SUB(CURDATE(), INTERVAL 1 YEAR)
    GROUP BY DATE(created_at), product_id;
END //
DELIMITER ;

-- Запланировать с MySQL Event Scheduler
SET GLOBAL event_scheduler = ON;

CREATE EVENT evt_refresh_mv_daily_sales
ON SCHEDULE EVERY 1 HOUR
STARTS CURRENT_TIMESTAMP
DO CALL refresh_mv_daily_sales();
```

## Кэширование результатов запросов

### PostgreSQL pg_hint_plan (кэширование запросов)

```sql
-- Установить pg_hint_plan для подсказок запросов
CREATE EXTENSION pg_hint_plan;

-- Принудительно использовать конкретный план выполнения
SELECT /*+ IndexScan(orders idx_orders_created_at) */
    *
FROM orders
WHERE created_at > '2024-01-01';
```

### Кэширование на уровне приложения

```python
# Python с кэшированием Redis
import redis
import json
import hashlib
from functools import wraps

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def cache_query(ttl=3600):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Генерировать ключ кэша из имени функции и аргументов
            key_data = f"{func.__name__}:{args}:{kwargs}"
            cache_key = hashlib.md5(key_data.encode()).hexdigest()

            # Попытаться получить из кэша
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)

            # Выполнить запрос и закэшировать результат
            result = func(*args, **kwargs)
            redis_client.setex(cache_key, ttl, json.dumps(result))
            return result
        return wrapper
    return decorator

@cache_query(ttl=300)  # Кэшировать на 5 минут
def get_daily_sales(date):
    cursor.execute("""
        SELECT product_id, total_amount
        FROM mv_daily_sales
        WHERE sale_date = %s
    """, (date,))
    return cursor.fetchall()
```

### Rails с Redis Cache

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, { url: ENV['REDIS_URL'] }

# В модели или сервисе
class SalesReport
  def self.daily_sales(date)
    Rails.cache.fetch("daily_sales:#{date}", expires_in: 5.minutes) do
      DailySale.where(sale_date: date).to_a
    end
  end

  def self.invalidate_cache(date)
    Rails.cache.delete("daily_sales:#{date}")
  end
end

# С зависимостями кэша
def monthly_summary(month)
  Rails.cache.fetch(["monthly_summary", month], expires_in: 1.hour) do
    Order.where(created_at: month.beginning_of_month..month.end_of_month)
         .group(:category_id)
         .sum(:amount)
  end
end
```

## Стратегии инвалидации кэша

### Истечение по времени

```python
# Простое кэширование с TTL
def get_cached_data(key, ttl=300):
    cached = redis_client.get(key)
    if cached:
        return json.loads(cached)

    data = fetch_from_database(key)
    redis_client.setex(key, ttl, json.dumps(data))
    return data
```

### Инвалидация на основе событий

```python
# Инвалидировать кэш при изменении данных
class OrderService:
    def create_order(self, order_data):
        order = Order.create(order_data)

        # Инвалидировать связанные кэши
        self.invalidate_caches(order)
        return order

    def invalidate_caches(self, order):
        date_key = order.created_at.strftime('%Y-%m-%d')

        # Инвалидировать конкретные ключи кэша
        redis_client.delete(f"daily_sales:{date_key}")
        redis_client.delete(f"product_stats:{order.product_id}")

        # Инвалидация по паттерну
        for key in redis_client.scan_iter(f"sales:*:{date_key}"):
            redis_client.delete(key)
```

### Write-Through кэш

```python
class CachedOrderRepository:
    def __init__(self, db, cache):
        self.db = db
        self.cache = cache

    def save(self, order):
        # Записать в базу данных
        self.db.save(order)

        # Обновить кэш немедленно
        cache_key = f"order:{order.id}"
        self.cache.set(cache_key, order.to_json())

        # Обновить агрегации
        self.update_aggregations(order)

    def get(self, order_id):
        cache_key = f"order:{order_id}"

        # Сначала попробовать кэш
        cached = self.cache.get(cache_key)
        if cached:
            return Order.from_json(cached)

        # Fallback на базу данных
        order = self.db.get(order_id)
        if order:
            self.cache.set(cache_key, order.to_json())
        return order
```

## Мониторинг материализованных представлений

### Статистика представлений

```sql
-- PostgreSQL: Проверить размеры материализованных представлений
SELECT
    schemaname,
    matviewname,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || matviewname)) AS total_size
FROM pg_matviews
ORDER BY pg_total_relation_size(schemaname || '.' || matviewname) DESC;

-- Проверить продолжительность обновления (требуется логирование)
SELECT
    schemaname,
    matviewname,
    last_refresh_start,
    last_refresh_end,
    last_refresh_end - last_refresh_start AS refresh_duration
FROM pg_matviews_refresh_log  -- Кастомная таблица логирования
ORDER BY last_refresh_end DESC;
```

### Коэффициент попадания в кэш

```python
# Мониторинг производительности Redis кэша
def get_cache_stats():
    info = redis_client.info()
    return {
        'keyspace_hits': info['keyspace_hits'],
        'keyspace_misses': info['keyspace_misses'],
        'hit_ratio': info['keyspace_hits'] / (info['keyspace_hits'] + info['keyspace_misses']),
        'used_memory': info['used_memory_human'],
        'connected_clients': info['connected_clients']
    }
```

### Правила алертинга

```yaml
# Алертинг Prometheus
groups:
  - name: materialized-views
    rules:
      - alert: MaterializedViewStale
        expr: time() - pg_matview_last_refresh_timestamp > 7200
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Материализованное представление {{ $labels.matview }} не обновлялось 2 часа"

      - alert: CacheHitRatioLow
        expr: redis_keyspace_hits_total / (redis_keyspace_hits_total + redis_keyspace_misses_total) < 0.8
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Коэффициент попадания в Redis кэш ниже 80%"

      - alert: MaterializedViewRefreshSlow
        expr: pg_matview_refresh_duration_seconds > 300
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Обновление материализованного представления занимает более 5 минут"
```

## Паттерн инкрементального обновления

```sql
-- Инкрементальное материализованное представление с отслеживанием изменений
CREATE TABLE mv_sales_incremental (
    sale_date DATE NOT NULL,
    product_id INT NOT NULL,
    total_quantity INT NOT NULL,
    total_amount DECIMAL(15,2) NOT NULL,
    order_count INT NOT NULL,
    last_order_id BIGINT NOT NULL,  -- Отслеживать последний обработанный заказ
    updated_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (sale_date, product_id)
);

-- Функция инкрементального обновления
CREATE OR REPLACE FUNCTION refresh_mv_sales_incremental()
RETURNS void AS $$
DECLARE
    last_id BIGINT;
BEGIN
    -- Получить последний обработанный ID заказа
    SELECT COALESCE(MAX(last_order_id), 0) INTO last_id
    FROM mv_sales_incremental;

    -- Вставить/обновить только новые данные
    INSERT INTO mv_sales_incremental (sale_date, product_id, total_quantity, total_amount, order_count, last_order_id)
    SELECT
        date_trunc('day', created_at)::date,
        product_id,
        SUM(quantity),
        SUM(amount),
        COUNT(*),
        MAX(id)
    FROM orders
    WHERE id > last_id
    GROUP BY 1, 2
    ON CONFLICT (sale_date, product_id)
    DO UPDATE SET
        total_quantity = mv_sales_incremental.total_quantity + EXCLUDED.total_quantity,
        total_amount = mv_sales_incremental.total_amount + EXCLUDED.total_amount,
        order_count = mv_sales_incremental.order_count + EXCLUDED.order_count,
        last_order_id = EXCLUDED.last_order_id,
        updated_at = NOW();
END;
$$ LANGUAGE plpgsql;
```

## Best practices

**Делать:**
- Использовать материализованные представления для сложных агрегаций и отчетов
- Создавать подходящие индексы на материализованных представлениях
- Использовать CONCURRENTLY обновление когда возможно для избежания блокировок
- Реализовать стратегию инвалидации кэша на основе паттернов изменения данных
- Мониторить коэффициенты попадания в кэш и корректировать TTL
- Рассмотреть инкрементальное обновление для больших датасетов
- Тестировать влияние обновления на production нагрузку

**Не делать:**
- Избыточно кэшировать волатильные данные с долгим TTL
- Забывать обновлять материализованные представления после изменения схемы
- Кэшировать данные, которые всегда уникальны для каждого запроса
- Игнорировать лимиты памяти кэша
- Использовать материализованные представления для требований реального времени
- Обновлять материализованные представления в часы пиковой нагрузки

## Переменные окружения

```bash
# Обновление материализованных представлений
MV_REFRESH_SCHEDULE="0 * * * *"
MV_REFRESH_CONCURRENT=true
MV_REFRESH_TIMEOUT=600

# Настройки Redis кэша
REDIS_URL=redis://localhost:6379/0
REDIS_CACHE_TTL=300
REDIS_MAX_MEMORY=1gb
REDIS_EVICTION_POLICY=allkeys-lru

# Поведение кэша
CACHE_ENABLED=true
CACHE_DEFAULT_TTL=300
CACHE_MAX_KEY_SIZE=1024
```

## Дополнительные ресурсы

- [PostgreSQL Materialized Views](https://www.postgresql.org/docs/current/sql-creatematerializedview.html)
- [pg_cron Extension](https://github.com/citusdata/pg_cron)
- [Redis Caching Patterns](https://redis.io/docs/manual/patterns/)
- [Cache-Aside Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
