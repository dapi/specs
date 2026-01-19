# SRS-010 Request Idempotency (Идемпотентность запросов)

## Определение

Идемпотентность - это свойство операции, при котором многократное выполнение операции с одинаковыми параметрами имеет тот же эффект, что и однократное выполнение.

Обязательно для:
* GET, HEAD, OPTIONS, TRACE методы (по стандарту HTTP)
* PUT, DELETE методы (обычно идемпотентны)
* Для POST при реализации специальной логики

## Проблема

```
Клиент отправляет запрос → Сетевой сбой → Клиент не получает ответ
Клиент повторяет запрос → Сервер обрабатывает ВТОРОЙ раз
→ Двойной списание средств / дублирование данных
```

## Idempotency Key

Решение: клиент генерирует уникальный ключ для каждой операции.

```
POST /payments HTTP/1.1
Idempotency-Key: 7ba7c8d5-9c4c-4c8c-bf9e-5d5f5f5f5f5f

{
  "amount": 1000,
  "currency": "USD",
  "account": "12345"
}
```

### Требования к ключу

* Уникальный для каждой операции
* Длиной минимум 8 символов, рекомендуется UUID v4
* Сохраняется на стороне клиента для возможности повтора
* Время жизни: минимум 24 часа, рекомендуется 7 дней

## Реализация на стороне сервера

### 1. Хранение ключей

Использовать для хранения сроком жизни:
* Redis с TTL
* Database с колонкой expires_at

```sql
CREATE TABLE idempotency_keys (
  key VARCHAR(255) PRIMARY KEY,
  response_body JSONB,
  response_status INTEGER,
  created_at TIMESTAMP,
  expires_at TIMESTAMP
);

CREATE INDEX ON idempotency_keys (expires_at);
```

### 2. Обработка запроса

```python
def handle_request(request):
    idempotency_key = request.headers.get('Idempotency-Key')

    if idempotency_key:
        # Проверяем, обрабатывали ли уже
        cached = storage.get(idempotency_key)

        if cached:
            # Возвращаем сохраненный ответ
            return cached.response_status, cached.response_body

    # Обрабатываем запрос
    response = process_business_logic(request)

    if idempotency_key:
        # Сохраняем ответ
        storage.set(idempotency_key, response, ttl=24h)

    return response
```

### 3. HTTP статусы

* **200 OK** - запрос обработан, ответ в теле
* **201 Created** - ресурс создан
* **409 Conflict** - запрос уже обрабатывается (concurrent request)
* **400 Bad Request** - невалидный idempotency key

## Конфигурация

Через переменные окружения:
```
IDEMPOTENCY_ENABLED=true
IDEMPOTENCY_KEY_TTL=86400  # в секундах, 24 часа
IDEMPOTENCY_STORAGE=redis  # или database
IDEMPOTENCY_KEY_MIN_LENGTH=8
```

## Метрики

```
idempotency_keys_stored (gauge)
idempotency_hits (counter)      # повторные запросы с тем же ключом
idempotency_misses (counter)    # новые запросы
idempotency_errors (counter)    # ошибки при работе с ключами
```

## Границы применения

Идемпотентность НЕ применяется к:
* Методу GET (идемпотентен по HTTP спецификации)
* Запросам без тела
* Health check запросам
* WebSocket соединениям

## Методы HTTP

### GET, HEAD, PUT, DELETE
Обычно идемпотентны по спецификации HTTP. Idempotency-Key не требуется.

### POST
Требует реализации с Idempotency-Key для критичных операций:
* Платежи
* Создание заказов
* Регистрация пользователей
* Отправка сообщений

### PATCH
Зависит от семантики:
```
PATCH /balance {"op": "add", "amount": 10}  # НЕ идемпотентен
PATCH /profile {"name": "John"}               # Идемпотентен
```

## Конкурентные запросы

Если два запроса с одним idempotency key приходят одновременно:

Вариант 1 (пессимистичный):
```
Второй запрос получает 409 Conflict
Первый запрос продолжает обработку
```

Вариант 2 (оптимистичный):
```
Второй запрос ждет завершения первого
Возвращает тот же результат
```

Рекомендуется вариант 1 для простоты реализации.

## Best practices

✅ **Делать**
* Генерировать ключ на клиенте до отправки
* Использовать UUID v4
* Валидировать формат ключа
* Хранить ответы с ограниченным TTL
* Возвращать одинаковый статус код
* Документировать идемпотентные endpoints

❌ **Не делать**
* Доверять генерацию ключа серверу
* Хранить ответы вечно
* Игнорировать дубликаты
* Применять к не-POST запросам без необходимости
* Разрешать слишком короткие ключи

## Пример реализации

```python
import uuid
import redis

class IdempotencyHandler:
    def __init__(self, redis_client):
        self.redis = redis_client

    def generate_key(self):
        return str(uuid.uuid4())

    def process(self, key, handler):
        if not self._is_valid_key(key):
            return 400, {"error": "Invalid idempotency key"}

        # Проверяем кэш
        cached = self.redis.get(f"idemp:{key}")
        if cached:
            return 200, json.loads(cached)

        # Ставим блокировку на время обработки
        if not self.redis.set(f"idemp:lock:{key}", "1", nx=True, ex=300):
            return 409, {"error": "Request being processed"}

        try:
            result = handler()
            self.redis.setex(f"idemp:{key}", 86400, json.dumps(result))
            return result
        finally:
            self.redis.delete(f"idemp:lock:{key}")

    def _is_valid_key(self, key):
        return len(key) >= 8 and re.match(r'^[a-zA-Z0-9-]+$', key)
```

## Дополнительные ресурсы

* [Stripe: Idempotent Requests](https://stripe.com/docs/api/idempotent_requests)
* [PayPal: API Idempotency](https://developer.paypal.com/docs/api/reference/api-requests/#http-request-headers)
* [HTTP Specification: Idempotent Methods](https://tools.ietf.org/html/rfc7231#section-4.2.2)
