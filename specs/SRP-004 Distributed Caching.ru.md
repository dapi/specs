# SRP-004 Distributed Caching

## Цель и задачи

Распределенный кэш обеспечивает хранение данных в памяти с низкой задержкой, высокой доступностью и возможностью масштабирования.

## Когда использовать

* Часто читаемые, редко изменяемые данные
* Результаты дорогостоящих вычислений
* Сессии пользователей
* Rate limiting состояния
* Временные блокировки

## Выбор решения

**Redis (рекомендуется)** - быстрый, многофункциональный, поддерживает TTL, pub/sub, streams
**Memcached** - очень быстрый, простой, но только key-value
**Hazelcast** - встроенные в JVM, rich data structures

## Ключи и именование

Формат: `<namespace>:<entity>:<id>:<field>`

Примеры:
```
app:users:123:name
app:products:456:price
app:config:feature_flags
```

## Стратегии кэширования

### Cache-Aside (Lazy Loading)
```python
def get_user(user_id):
    key = f"app:users:{user_id}"
    cached = redis.get(key)
    if cached:
        return json.loads(cached)
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    redis.setex(key, 3600, json.dumps(user))
    return user
```

### Write-Through
```python
def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = %s", user_id, data)
    key = f"app:users:{user_id}"
    redis.setex(key, 3600, json.dumps(data))
    return data
```

### TTL и Время жизни
```
# Данные пользователя - 1 час
redis.setex("app:users:123", 3600, data)

# Сессия - 24 часа
redis.setex("app:session:abc", 86400, data)

# Feature flags - 5 минут
redis.setex("app:config:flags", 300, data)
```

## Распределенные блокировки

```python
def process_payment(order_id):
    lock_key = f"lock:payment:{order_id}"
    lock_acquired = redis.set(lock_key, "1", nx=True, ex=60)
    if not lock_acquired:
        raise Error("Payment already processing")
    try:
        process(order_id)
    finally:
        redis.delete(lock_key)
```

## Конфигурация

```
CACHE_ENABLED=true
CACHE_TYPE=redis
CACHE_URL=redis://cache:6379/0
CACHE_TTL_DEFAULT=3600
CACHE_TTL_SESSION=86400
CACHE_TTL_CONFIG=300

CACHE_POOL_MIN_SIZE=5
CACHE_POOL_MAX_SIZE=20
CACHE_POOL_TIMEOUT=5000
```

## Метрики

```
cache_hits_total (counter)
cache_misses_total (counter)
cache_evictions_total (counter)
cache_size_bytes (gauge)
cache_latency_seconds (histogram)
cache_errors_total (counter)
```

## Проблемы и решения

### Cache Stampede
Решение: Probabilistic early expiration или Mutex для пересоздания

### Холодный кэш (Cold Start)
Решение: Cache warming при старте, Redis persistence

### Размер кэша
Решение: maxmemory в Redis, LRU eviction policy, мониторинг на 80% использования

## Best practices

✅ **Делать**
* Устанавливать TTL на все ключи
* Именовать ключи с namespace
* Мониторить hit rate (цель > 80%)
* Использовать connection pooling
* Включить persistence для Redis

❌ **Не делать**
* Хранить сессии без TTL
* Кэшировать вечно
* Использовать кэш как основное хранилище
* Кэшировать все подряд
* Игнорировать метрики

## Дополнительные ресурсы

* [Redis Best Practices](https://redis.io/topics/best-practices)
* [Caching Strategies](https://docs.microsoft.com/en-us/azure/architecture/best-practices/caching)
* [Cache-Aside Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
