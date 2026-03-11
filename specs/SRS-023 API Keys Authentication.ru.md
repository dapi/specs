# SRS-023 API Keys Authentication (Аутентификация по API-ключам)

## Определение

API Keys — общие секреты, передаваемые в каждом запросе (обычно через HTTP-заголовок) для идентификации и аутентификации вызывающего сервиса. Простейший механизм аутентификации — не требует PKI или auth-сервера.

## Когда использовать

| Сценарий | Рекомендация |
|----------|-------------|
| Простые интеграции сторонних сервисов | ✅ Приемлемо |
| Совместимость с legacy-системами | ✅ Приемлемо |
| Среды разработки и тестирования | ✅ Приемлемо |
| Внутренний service-to-service | ❌ Использовать mTLS или JWT |
| Высокие требования к безопасности | ❌ Использовать mTLS |
| Гранулярный контроль доступа | ❌ Использовать OAuth 2.0 |

## Формат ключа

Ключи должны быть:
* Криптографически случайными — минимум 256 бит энтропии
* Непредсказуемыми — не производными от имени сервиса или временной метки
* С префиксом для удобной идентификации в логах: `prod_`, `test_`, `sk_live_`

```python
import secrets

def generate_api_key(prefix: str = "sk") -> str:
    # 32 байта = 256 бит энтропии, base64url-кодировка
    random_part = secrets.token_urlsafe(32)
    return f"{prefix}_{random_part}"

# Пример вывода: sk_Xt4mP9qR2v...
```

## Передача

Ключи ДОЛЖНЫ передаваться в HTTP-заголовке, никогда — в URL:

```bash
# Правильно
curl -H "X-API-Key: sk_Xt4mP9qR2v..." https://api.example.com/data

# Неправильно — ключ попадает в логи сервера и историю браузера
curl "https://api.example.com/data?api_key=sk_Xt4mP9qR2v..."
```

## Хранение

Ключи ДОЛЖНЫ храниться в хешированном виде, никогда — в plaintext:

```python
import hashlib
import hmac
import os
from pathlib import Path

def load_api_key_pepper() -> str:
    path = os.getenv("SERVICE_AUTH_API_KEY_PEPPER_PATH", "/etc/secrets/api_key_pepper")
    return Path(path).read_text().strip()

def hash_api_key(raw_key: str) -> str:
    """Хешировать ключ для хранения. Использовать SHA-256 с pepper."""
    pepper = load_api_key_pepper()
    return hashlib.sha256(f"{pepper}{raw_key}".encode()).hexdigest()

def verify_api_key(raw_key: str, stored_hash: str) -> bool:
    """Timing-safe сравнение для защиты от timing-атак."""
    expected = hash_api_key(raw_key)
    return hmac.compare_digest(expected, stored_hash)
```

## Реализация

### Flask Decorator

```python
from functools import wraps
from flask import request, jsonify, g

def require_api_key(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        raw_key = request.headers.get("X-API-Key")

        if not raw_key:
            return jsonify({"error": "API key required"}), 401

        service = lookup_service_by_key(raw_key)
        if service is None:
            return jsonify({"error": "Invalid API key"}), 403

        g.service_name = service.name
        g.service_id = service.id
        return f(*args, **kwargs)

    return decorated_function

def lookup_service_by_key(raw_key: str):
    """Найти сервис по хешу ключа. Возвращает None если не найден."""
    key_hash = hash_api_key(raw_key)
    return Service.query.filter_by(
        api_key_hash=key_hash,
        is_active=True
    ).first()

@app.route("/api/data")
@require_api_key
def get_data():
    return jsonify({"service": g.service_name, "data": "..."})
```

### FastAPI Middleware

```python
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()

@app.middleware("http")
async def api_key_middleware(request: Request, call_next):
    if request.url.path in ["/health", "/live", "/ready"]:
        return await call_next(request)

    raw_key = request.headers.get("X-API-Key")
    if not raw_key:
        raise HTTPException(status_code=401, detail="API key required")

    service = await lookup_service_by_key_async(raw_key)
    if service is None:
        raise HTTPException(status_code=403, detail="Invalid API key")

    request.state.service_name = service.name
    return await call_next(request)
```

## Ротация ключей

API-ключи ДОЛЖНЫ поддерживать ротацию без остановки сервиса:

```python
# Схема БД поддерживает несколько активных ключей на сервис
# для rolling-ротации без даунтайма
class ApiKey(Base):
    __tablename__ = "api_keys"
    id = Column(UUID, primary_key=True)
    service_id = Column(UUID, ForeignKey("services.id"), nullable=False)
    key_hash = Column(String, nullable=False, unique=True)
    key_prefix = Column(String(8))   # например "sk_Xt4m" для идентификации
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime)
    expires_at = Column(DateTime)    # Опциональный жёсткий срок истечения
    last_used_at = Column(DateTime)  # Для аудита старых ключей
```

### Процедура ротации

```
1. Генерировать новый ключ → выдать сервису
2. Сервис деплоит с новым ключом (оба ключа активны)
3. Убедиться, что новый ключ работает в production
4. Деактивировать старый ключ (set is_active=False)
5. Удалить старый ключ после 7-дневного grace-периода
```

## Конфигурация

```
SERVICE_AUTH_API_KEY_HEADER=X-API-Key
SERVICE_AUTH_API_KEY_PREFIX=sk
SERVICE_AUTH_API_KEY_ROTATION_ENABLED=true
SERVICE_AUTH_API_KEY_MAX_ACTIVE_KEYS=2          # На сервис во время ротации
SERVICE_AUTH_API_KEY_PEPPER_PATH=/etc/secrets/api_key_pepper
```

## Мониторинг

```
service_api_key_requests_total{result="success|failure"} (counter)
service_api_key_invalid_attempts_total{service="name"} (counter)
service_api_key_active_keys_total{service="name"} (gauge)
service_api_key_last_rotation_timestamp{service="name"} (gauge)
```

Алерт на: более 5 невалидных попыток за 1 минуту с одного IP (возможный brute force).

## Best Practices

✅ **Делать**
* Генерировать ключи с минимум 256 битами энтропии
* Хранить только хеш, никогда — сырой ключ
* Использовать timing-safe сравнение (`hmac.compare_digest`)
* Поддерживать ротацию ключей с grace-периодом
* Логировать использование ключа с `key_prefix` (не полный ключ)
* Устанавливать сроки истечения для ключей внешних интеграций
* Передавать только через HTTPS

❌ **Не делать**
* Использовать API-ключи для внутреннего service-to-service
* Хранить сырые ключи в базе данных или логах
* Принимать ключи в URL query-параметрах
* Использовать один ключ для нескольких сервисов
* Оставлять ключи бессрочными без ротации

## Плюсы и минусы

**Плюсы:**
* Простота реализации и использования
* Нет инфраструктурных зависимостей (не нужны PKI, auth-сервер)
* Низкие накладные расходы на запрос
* Удобно выдавать внешним партнёрам

**Минусы:**
* Ключ не истекает автоматически (нужен явный отзыв)
* Нет встроенной системы scopes или разрешений
* Передаётся в каждом запросе (бо́льшая поверхность воздействия)
* Отслеживание использования требует кастомной реализации

## Дополнительные ресурсы

* [OWASP API Security — Broken Authentication](https://owasp.org/www-project-api-security/)
* [RFC 7235 — HTTP Authentication](https://tools.ietf.org/html/rfc7235)
* [HashiCorp Vault — управление секретами](https://www.vaultproject.io/)
