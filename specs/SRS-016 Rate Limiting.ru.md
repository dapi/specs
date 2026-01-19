# SRS-016 Rate Limiting (Ограничение частоты запросов)

Rate limiting - это механизм контроля скорости обработки запросов, который ограничивает количество запросов, которые клиент может сделать к сервису за определенный период времени.

## Зачем нужен Rate Limiting?

### Защита от перегрузки
Предотвращает перегрузку сервиса большим количеством запросов, которые он не может обработать.

### Предотвращение DoS-атак
Защищает от злонамеренных атак, направленных на недоступность сервиса.

### Обеспечение честного использования
Гарантирует, что ресурсы распределяются равномерно между всеми пользователями.

### Контроль затрат
Ограничивает использование платных API и ресурсов.

## Алгоритмы Rate Limiting

### 1. Fixed Window Counter (Фиксированное окно)

Простой счетчик запросов в фиксированном временном окне.

```
Окно: 1 минута
Лимит: 100 запросов

12:00:00 - 12:00:59: 100 запросов ✅
12:01:00 - 12:01:59: 100 запросов ✅
```

**Проблема**: "Всплеск в конце окна" - клиент может сделать 100 запросов в 12:00:59 и еще 100 в 12:01:00 (200 запросов за 2 секунды).

### 2. Sliding Window Log (Скользящее окно с логом)

Хранит временную метку каждого запроса.

```python
class SlidingWindowLog:
    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size
        self.requests = []

    def is_allowed(self, client_id):
        now = time.time()
        # Удаляем старые запросы
        self.requests = [req for req in self.requests if now - req < self.window_size]

        if len(self.requests) < self.limit:
            self.requests.append(now)
            return True
        return False
```

**Плюсы**: Точный учет
**Минусы**: Большой объем хранения

### 3. Sliding Window Counter (Скользящее окно с счетчиком)

Комбинация Fixed Window и Sliding Window Log.

```
Текущее окно: 12:00:00 - 12:01:00
Предыдущее окно: 11:59:00 - 12:00:00

Запрос в 12:00:45:
Запросов в текущем окне: 80
Запросов в предыдущем окне: 100
Процент времени в текущем окне: 75%

Оценка = 80 + (100 * 0.75) = 155 запросов
Лимит = 100 запросов
Результат: BLOCKED ❌
```

**Плюсы**: Хороший баланс точности и эффективности
**Минусы**: Может быть небольшое превышение лимита

### 4. Token Bucket (Ведро токенов)

Алгоритм с токенами, которые пополняются со временем.

```python
class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity        # Максимум токенов
        self.refill_rate = refill_rate  # Токенов в секунду
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

**Плюсы**: Позволяет всплески запросов до `capacity`
**Минусы**: Может быть сложно настроить optimal capacity

### 5. Leaky Bucket (Протекающее ведро)

Запросы "течут" через ведро с фиксированной скоростью.

```
Запросы → [Ведро] → Обработка (фиксированная скорость)
```

**Плюсы**: Гарантированная скорость обработки
**Минусы**: Не позволяет всплески

## Стратегии ограничения

### По IP-адресу
```
Ограничение по IP: 1000 запросов в час с одного IP
```

### По пользователю
```
Ограничение по API key: 10000 запросов в день
```

### По endpoint
```
GET /api/users: 100 запросов в минуту
POST /api/users: 10 запросов в минуту
```

### По сервису (глобальное)
```
Всего запросов к сервису: 100000 в минуту
```

## HTTP заголовки ответа

Сервис должен возвращать заголовки:

```
X-RateLimit-Limit: 100          # Лимит запросов
X-RateLimit-Remaining: 85       # Осталось запросов
X-RateLimit-Reset: 1640000000  # Unix timestamp когда лимит сбросится
Retry-After: 3600               # Секунд до повторной попытки (при 429)
```

## HTTP статусы

- **200 OK** - запрос успешен
- **429 Too Many Requests** - лимит превышен
- **503 Service Unavailable** - сервис перегружен (circuit breaker)

## Реализация на Redis

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

        # Удаляем старые запросы
        self.redis.zremrangebyscore(key, '-inf', window_start)

        # Получаем количество запросов в окне
        request_count = self.redis.zcard(key)

        if request_count < limit:
            # Добавляем текущий запрос
            self.redis.zadd(key, {str(now): now})
            # Устанавливаем TTL
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

## Конфигурация

Через переменные окружения:
```
RATE_LIMIT_ENABLED=true
RATE_LIMIT_ALGORITHM=sliding_window
RATE_LIMIT_DEFAULT=1000  # запросов
RATE_LIMIT_WINDOW=3600   # секунд (1 час)
RATE_LIMIT_PER_IP=100
RATE_LIMIT_PER_USER=10000
RATE_LIMIT_BURST=50      # для token bucket
```

## Обработка превышения лимита

### Стратегии
1. **Жесткий отказ** - немедленный ответ 429
2. **Очередь** - запросы ставятся в очередь (опасно!)
3. **Prioritization** - высокоприоритетные запросы проходят

### Circuit Breaker + Rate Limiter
```python
def handle_request(request):
    # 1. Проверяем rate limit
    if not rate_limiter.is_allowed(request.client_id):
        return 429, "Rate limit exceeded"

    # 2. Проверяем circuit breaker
    if circuit_breaker.is_open():
        return 503, "Service unavailable"

    # 3. Обрабатываем запрос
    return process_request(request)
```

## Best practices

✅ **Делать**
- Возвращать четкие HTTP заголовки
- Документировать лимиты в API documentation
- Использовать разные лимиты для разных пользователей
- Мониторить количество отклоненных запросов
- Предоставлять способ увеличить лимит
- Тестировать поведение при превышении лимита

❌ **Не делать**
- Возвращать 403 Forbidden (это про авторизацию, не rate limit)
- Сбрасывать лимит сразу в начале нового окна
- Использовать один лимит для всех endpoint'ов
- Игнорировать burst трафика
- Блокировать навсегда (всегда давайте TTL)

## Метрики

```
ratelimit_requests_total (counter)        # всего запросов
ratelimit_requests_allowed (counter)      # разрешенных
ratelimit_requests_blocked (counter)      # отклоненных
ratelimit_current_usage (gauge)           # текущее использование
ratelimit_avg_response_time (histogram)   # время проверки лимита
```

## Стоимость реализации

| Алгоритм | Сложность | Память | Точность |
|----------|-----------|--------|----------|
| Fixed Window | Низкая | Низкая | Средняя |
| Sliding Window Log | Средняя | Высокая | Высокая |
| Sliding Window Counter | Средняя | Средняя | Высокая |
| Token Bucket | Низкая | Низкая | Высокая |
| Leaky Bucket | Средняя | Средняя | Высокая |

## Дополнительные ресурсы

* [IETF: Rate Limit Fields for HTTP](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)
* [Stripe API: Rate Limits](https://stripe.com/docs/rate-limits)
* [GitHub API: Rate Limiting](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)
* [CloudFlare: Rate Limiting продукта](https://www.cloudflare.com/rate-limiting/)
