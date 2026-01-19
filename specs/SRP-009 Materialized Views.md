# SRP-009 Materialized Views & Caching

## Definition

Materialized views are database objects that store the result of a query physically on disk, unlike regular views which execute the query each time. Combined with caching strategies, they significantly improve read performance for complex queries and aggregations.

## PostgreSQL Materialized Views

### Creating Materialized Views

```sql
-- Basic materialized view
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

-- With no data (create structure, populate later)
CREATE MATERIALIZED VIEW mv_monthly_report AS
SELECT
    date_trunc('month', created_at) AS month,
    category_id,
    SUM(revenue) AS total_revenue
FROM sales
GROUP BY 1, 2
WITH NO DATA;

-- Populate later
REFRESH MATERIALIZED VIEW mv_monthly_report;
```

### Indexing Materialized Views

```sql
-- Create indexes on materialized view for faster lookups
CREATE UNIQUE INDEX idx_mv_daily_sales_date_product
    ON mv_daily_sales (sale_date, product_id);

CREATE INDEX idx_mv_daily_sales_product
    ON mv_daily_sales (product_id);

CREATE INDEX idx_mv_daily_sales_amount
    ON mv_daily_sales (total_amount DESC);
```

### Refreshing Materialized Views

```sql
-- Full refresh (blocks reads during refresh)
REFRESH MATERIALIZED VIEW mv_daily_sales;

-- Concurrent refresh (requires unique index, no blocking)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_sales;

-- Check when materialized view was last refreshed
SELECT
    schemaname,
    matviewname,
    hasindexes,
    ispopulated,
    definition
FROM pg_matviews
WHERE matviewname = 'mv_daily_sales';
```

### Automatic Refresh with pg_cron

```sql
-- Install pg_cron extension
CREATE EXTENSION pg_cron;

-- Schedule hourly refresh
SELECT cron.schedule(
    'refresh_mv_daily_sales',
    '0 * * * *',  -- Every hour
    'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_sales'
);

-- Schedule daily refresh at 2 AM
SELECT cron.schedule(
    'refresh_mv_monthly_report',
    '0 2 * * *',
    'REFRESH MATERIALIZED VIEW mv_monthly_report'
);

-- List scheduled jobs
SELECT * FROM cron.job;

-- Remove scheduled job
SELECT cron.unschedule('refresh_mv_daily_sales');
```

### Trigger-Based Refresh

```sql
-- Function to refresh materialized view
CREATE OR REPLACE FUNCTION refresh_mv_on_change()
RETURNS TRIGGER AS $$
BEGIN
    -- Refresh asynchronously using pg_notify
    PERFORM pg_notify('refresh_mv', 'mv_daily_sales');
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Trigger on source table
CREATE TRIGGER trg_refresh_mv_daily_sales
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH STATEMENT
    EXECUTE FUNCTION refresh_mv_on_change();

-- Background worker listens and refreshes
-- (Implemented in application code)
```

## MySQL Materialized Views (Simulated)

MySQL doesn't have native materialized views, but you can simulate them:

```sql
-- Create summary table (acts as materialized view)
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

-- Refresh procedure
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

-- Schedule with MySQL Event Scheduler
SET GLOBAL event_scheduler = ON;

CREATE EVENT evt_refresh_mv_daily_sales
ON SCHEDULE EVERY 1 HOUR
STARTS CURRENT_TIMESTAMP
DO CALL refresh_mv_daily_sales();
```

## Query Result Caching

### PostgreSQL pg_hint_plan (Query Caching)

```sql
-- Install pg_hint_plan for query hints
CREATE EXTENSION pg_hint_plan;

-- Force specific execution plan
SELECT /*+ IndexScan(orders idx_orders_created_at) */
    *
FROM orders
WHERE created_at > '2024-01-01';
```

### Application-Level Caching

```python
# Python with Redis caching
import redis
import json
import hashlib
from functools import wraps

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def cache_query(ttl=3600):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Generate cache key from function name and arguments
            key_data = f"{func.__name__}:{args}:{kwargs}"
            cache_key = hashlib.md5(key_data.encode()).hexdigest()

            # Try to get from cache
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)

            # Execute query and cache result
            result = func(*args, **kwargs)
            redis_client.setex(cache_key, ttl, json.dumps(result))
            return result
        return wrapper
    return decorator

@cache_query(ttl=300)  # Cache for 5 minutes
def get_daily_sales(date):
    cursor.execute("""
        SELECT product_id, total_amount
        FROM mv_daily_sales
        WHERE sale_date = %s
    """, (date,))
    return cursor.fetchall()
```

### Rails with Redis Cache

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, { url: ENV['REDIS_URL'] }

# In model or service
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

# With cache dependencies
def monthly_summary(month)
  Rails.cache.fetch(["monthly_summary", month], expires_in: 1.hour) do
    Order.where(created_at: month.beginning_of_month..month.end_of_month)
         .group(:category_id)
         .sum(:amount)
  end
end
```

## Cache Invalidation Strategies

### Time-Based Expiration

```python
# Simple TTL-based caching
def get_cached_data(key, ttl=300):
    cached = redis_client.get(key)
    if cached:
        return json.loads(cached)

    data = fetch_from_database(key)
    redis_client.setex(key, ttl, json.dumps(data))
    return data
```

### Event-Based Invalidation

```python
# Invalidate cache when data changes
class OrderService:
    def create_order(self, order_data):
        order = Order.create(order_data)

        # Invalidate related caches
        self.invalidate_caches(order)
        return order

    def invalidate_caches(self, order):
        date_key = order.created_at.strftime('%Y-%m-%d')

        # Invalidate specific cache keys
        redis_client.delete(f"daily_sales:{date_key}")
        redis_client.delete(f"product_stats:{order.product_id}")

        # Pattern-based invalidation
        for key in redis_client.scan_iter(f"sales:*:{date_key}"):
            redis_client.delete(key)
```

### Write-Through Cache

```python
class CachedOrderRepository:
    def __init__(self, db, cache):
        self.db = db
        self.cache = cache

    def save(self, order):
        # Write to database
        self.db.save(order)

        # Update cache immediately
        cache_key = f"order:{order.id}"
        self.cache.set(cache_key, order.to_json())

        # Update aggregations
        self.update_aggregations(order)

    def get(self, order_id):
        cache_key = f"order:{order_id}"

        # Try cache first
        cached = self.cache.get(cache_key)
        if cached:
            return Order.from_json(cached)

        # Fallback to database
        order = self.db.get(order_id)
        if order:
            self.cache.set(cache_key, order.to_json())
        return order
```

## Monitoring Materialized Views

### View Statistics

```sql
-- PostgreSQL: Check materialized view sizes
SELECT
    schemaname,
    matviewname,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || matviewname)) AS total_size
FROM pg_matviews
ORDER BY pg_total_relation_size(schemaname || '.' || matviewname) DESC;

-- Check refresh duration (requires logging)
SELECT
    schemaname,
    matviewname,
    last_refresh_start,
    last_refresh_end,
    last_refresh_end - last_refresh_start AS refresh_duration
FROM pg_matviews_refresh_log  -- Custom logging table
ORDER BY last_refresh_end DESC;
```

### Cache Hit Ratio

```python
# Monitor Redis cache performance
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

### Alerting Rules

```yaml
# Prometheus alerting
groups:
  - name: materialized-views
    rules:
      - alert: MaterializedViewStale
        expr: time() - pg_matview_last_refresh_timestamp > 7200
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Materialized view {{ $labels.matview }} not refreshed in 2 hours"

      - alert: CacheHitRatioLow
        expr: redis_keyspace_hits_total / (redis_keyspace_hits_total + redis_keyspace_misses_total) < 0.8
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Redis cache hit ratio below 80%"

      - alert: MaterializedViewRefreshSlow
        expr: pg_matview_refresh_duration_seconds > 300
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Materialized view refresh taking more than 5 minutes"
```

## Incremental Refresh Pattern

```sql
-- Incremental materialized view with change tracking
CREATE TABLE mv_sales_incremental (
    sale_date DATE NOT NULL,
    product_id INT NOT NULL,
    total_quantity INT NOT NULL,
    total_amount DECIMAL(15,2) NOT NULL,
    order_count INT NOT NULL,
    last_order_id BIGINT NOT NULL,  -- Track last processed order
    updated_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (sale_date, product_id)
);

-- Incremental refresh function
CREATE OR REPLACE FUNCTION refresh_mv_sales_incremental()
RETURNS void AS $$
DECLARE
    last_id BIGINT;
BEGIN
    -- Get last processed order ID
    SELECT COALESCE(MAX(last_order_id), 0) INTO last_id
    FROM mv_sales_incremental;

    -- Insert/update only new data
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

## Best Practices

**Do:**
- Use materialized views for complex aggregations and reports
- Create appropriate indexes on materialized views
- Use CONCURRENTLY refresh when possible to avoid blocking
- Implement cache invalidation strategy based on data change patterns
- Monitor cache hit ratios and adjust TTLs accordingly
- Consider incremental refresh for large datasets
- Test refresh impact on production workload

**Don't:**
- Over-cache volatile data with long TTLs
- Forget to refresh materialized views after schema changes
- Cache data that's always unique per request
- Ignore cache memory limits
- Use materialized views for real-time data requirements
- Refresh materialized views during peak traffic hours

## Environment Variables

```bash
# Materialized view refresh
MV_REFRESH_SCHEDULE="0 * * * *"
MV_REFRESH_CONCURRENT=true
MV_REFRESH_TIMEOUT=600

# Redis cache settings
REDIS_URL=redis://localhost:6379/0
REDIS_CACHE_TTL=300
REDIS_MAX_MEMORY=1gb
REDIS_EVICTION_POLICY=allkeys-lru

# Cache behavior
CACHE_ENABLED=true
CACHE_DEFAULT_TTL=300
CACHE_MAX_KEY_SIZE=1024
```

## Additional Resources

- [PostgreSQL Materialized Views](https://www.postgresql.org/docs/current/sql-creatematerializedview.html)
- [pg_cron Extension](https://github.com/citusdata/pg_cron)
- [Redis Caching Patterns](https://redis.io/docs/manual/patterns/)
- [Cache-Aside Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
