# SRP-004 Distributed Caching

## Goals

Distributed cache provides in-memory data storage with low latency, high availability, and scalability capabilities.

## When to use

* Frequently read, rarely changed data
* Results of expensive computations
* User sessions
* Rate limiting state
* Temporary locks

## Choosing a solution

**Redis (recommended)** - fast, feature-rich, supports TTL, pub/sub, streams
**Memcached** - very fast, simple, but only key-value
**Hazelcast** - embedded in JVM, rich data structures

## Keys and naming

Format: `<namespace>:<entity>:<id>:<field>`

Examples:
```
app:users:123:name
app:products:456:price
app:config:feature_flags
```

## Caching strategies

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

### TTL and lifetime
```
# User data - 1 hour
redis.setex("app:users:123", 3600, data)

# Session - 24 hours
redis.setex("app:session:abc", 86400, data)

# Feature flags - 5 minutes
redis.setex("app:config:flags", 300, data)
```

## Distributed locks

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

## Configuration

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

## Metrics

```
cache_hits_total (counter)
cache_misses_total (counter)
cache_evictions_total (counter)
cache_size_bytes (gauge)
cache_latency_seconds (histogram)
cache_errors_total (counter)
```

## Problems and solutions

### Cache Stampede
Solution: Probabilistic early expiration or Mutex for rebuilding

### Cold Start
Solution: Cache warming at startup, Redis persistence

### Cache size
Solution: maxmemory in Redis, LRU eviction policy, monitor at 80% usage

## Best practices

✅ **Do**
* Set TTL on all keys
* Name keys with namespace
* Monitor hit rate (target > 80%)
* Use connection pooling
* Enable persistence for Redis

❌ **Don't**
* Store sessions without TTL
* Cache forever
* Use cache as primary storage
* Cache everything indiscriminately
* Ignore metrics

## Additional resources

* [Redis Best Practices](https://redis.io/topics/best-practices)
* [Caching Strategies](https://docs.microsoft.com/en-us/azure/architecture/best-practices/caching)
* [Cache-Aside Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
