# SRS-016 Rate Limiting

Rate limiting is a request processing speed control mechanism that limits the number of requests a client can make to a service within a specified period of time.

## Why Rate Limiting is needed?

### Protection from overload
Prevents service overload with a large number of requests it cannot handle.

### DoS attack prevention
Protects against malicious attacks aimed at service unavailability.

### Ensuring fair usage
Guarantees that resources are distributed evenly among all users.

### Cost control
Limits the use of paid APIs and resources.

## Rate Limiting algorithms

### 1. Fixed Window Counter (Fixed window)

Simple counter of requests in a fixed time window.

```
Window: 1 minute
Limit: 100 requests

12:00:00 - 12:00:59: 100 requests ✅
12:01:00 - 12:01:59: 100 requests ✅
```

**Problem**: "Burst at end of window" - client can make 100 requests at 12:00:59 and another 100 at 12:01:00 (200 requests in 2 seconds).

### 2. Sliding Window Log (Sliding window with log)

Stores timestamp of each request.

```python
class SlidingWindowLog:
    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size
        self.requests = []

    def is_allowed(self, client_id):
        now = time.time()
        # Remove old requests
        self.requests = [req for req in self.requests if now - req < self.window_size]

        if len(self.requests) < self.limit:
            self.requests.append(now)
            return True
        return False
```

**Pros**: Accurate accounting
**Cons**: Large storage volume

### 3. Sliding Window Counter (Sliding window with counter)

Combination of Fixed Window and Sliding Window Log.

```
Current window: 12:00:00 - 12:01:00
Previous window: 11:59:00 - 12:00:00

Request at 12:00:45:
Requests in current window: 80
Requests in previous window: 100
Percentage of time in current window: 75%

Estimate = 80 + (100 * 0.75) = 155 requests
Limit = 100 requests
Result: BLOCKED ❌
```

**Pros**: Good balance of accuracy and efficiency
**Cons**: Slight limit overrun possible

### 4. Token Bucket (Token bucket)

Algorithm with tokens that refill over time.

```python
class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity        # Maximum tokens
        self.refill_rate = refill_rate  # Tokens per second
        self.tokens = capacity
        self.last_refill = time.time()

    def is_allowed(self):
        self._refill()

        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False

    def _refill(self):
        now = time.time()
        elapsed = now - self.last_refill
        new_tokens = elapsed * self.refill_rate

        self.tokens = min(self.capacity, self.tokens + new_tokens)
        self.last_refill = now
```

**Pros**: Allows request bursts up to `capacity`
**Cons**: Can be difficult to configure optimal capacity

### 5. Leaky Bucket (Leaking bucket)

Requests "leak" through a bucket at a fixed rate.

```
Requests → [Bucket] → Processing (fixed rate)
```

**Pros**: Guaranteed processing rate
**Cons**: Doesn't allow bursts

## Limiting strategies

### By IP address
```
Limit by IP: 1000 requests per hour from one IP
```

### By user
```
Limit by API key: 10000 requests per day
```

### By endpoint
```
GET /api/users: 100 requests per minute
POST /api/users: 10 requests per minute
```

### By service (global)
```
Total requests to service: 100000 per minute
```

## HTTP response headers

Service must return headers:

```
X-RateLimit-Limit: 100          # Request limit
X-RateLimit-Remaining: 85       # Requests remaining
X-RateLimit-Reset: 1640000000  # Unix timestamp when limit resets
Retry-After: 3600               # Seconds until retry (on 429)
```

## HTTP status codes

- **200 OK** - request successful
- **429 Too Many Requests** - limit exceeded
- **503 Service Unavailable** - service overloaded (circuit breaker)

## Implementation in Redis

```python
import redis
import time

class RedisRateLimiter:
    def __init__(self, redis_client, key_prefix='ratelimit'):
        self.redis = redis_client
        self.key_prefix = key_prefix

    def is_allowed(self, client_id, limit, window_size):
        key = f"{self.key_prefix}:{client_id}"
        now = int(time.time() * 1000)  # milliseconds
        window_start = now - (window_size * 1000)

        # Remove old requests
        self.redis.zremrangebyscore(key, '-inf', window_start)

        # Get number of requests in window
        request_count = self.redis.zcard(key)

        if request_count < limit:
            # Add current request
            self.redis.zadd(key, {str(now): now})
            # Set TTL
            self.redis.expire(key, window_size)
            return True

        return False

    def get_status(self, client_id, limit, window_size):
        key = f"{self.key_prefix}:{client_id}"
        now = int(time.time() * 1000)
        window_start = now - (window_size * 1000)

        self.redis.zremrangebyscore(key, '-inf', window_start)
        request_count = self.redis.zcard(key)

        return {
            'limit': limit,
            'remaining': max(0, limit - request_count),
            'used': request_count
        }
```

## Configuration

Via environment variables:
```
RATE_LIMIT_ENABLED=true
RATE_LIMIT_ALGORITHM=sliding_window
RATE_LIMIT_DEFAULT=1000  # requests
RATE_LIMIT_WINDOW=3600   # seconds (1 hour)
RATE_LIMIT_PER_IP=100
RATE_LIMIT_PER_USER=10000
RATE_LIMIT_BURST=50      # for token bucket
```

## Handling limit exceeded

### Strategies
1. **Hard reject** - immediate 429 response
2. **Queue** - requests are queued (dangerous!)
3. **Prioritization** - high-priority requests pass

### Circuit Breaker + Rate Limiter
```python
def handle_request(request):
    # 1. Check rate limit
    if not rate_limiter.is_allowed(request.client_id):
        return 429, "Rate limit exceeded"

    # 2. Check circuit breaker
    if circuit_breaker.is_open():
        return 503, "Service unavailable"

    # 3. Process request
    return process_request(request)
```

## Best practices

✅ **Do**
- Return clear HTTP headers
- Document limits in API documentation
- Use different limits for different users
- Monitor number of rejected requests
- Provide way to increase limit
- Test limit exceeded behavior

❌ **Don't**
- Return 403 Forbidden (this is about authorization, not rate limit)
- Reset limit immediately at new window start
- Use one limit for all endpoints
- Ignore burst traffic
- Block forever (always give TTL)

## Metrics

```
ratelimit_requests_total (counter)        # total requests
ratelimit_requests_allowed (counter)      # allowed
ratelimit_requests_blocked (counter)      # blocked
ratelimit_current_usage (gauge)           # current usage
ratelimit_avg_response_time (histogram)   # check time
```

## Implementation cost

| Algorithm | Complexity | Memory | Accuracy |
|----------|-----------|--------|----------|
| Fixed Window | Low | Low | Medium |
| Sliding Window Log | Medium | High | High |
| Sliding Window Counter | Medium | Medium | High |
| Token Bucket | Low | Low | High |
| Leaky Bucket | Medium | Medium | High |

## Additional resources

* [IETF: Rate Limit Fields for HTTP](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)
* [Stripe API: Rate Limits](https://stripe.com/docs/rate-limits)
* [GitHub API: Rate Limiting](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)
* [CloudFlare: Rate Limiting product](https://www.cloudflare.com/rate-limiting/)
