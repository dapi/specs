# SRS-021 JWT Service Tokens (JWT-токены для сервисов)

**Статус**: APPROVED
**Связанные**: [SRS-019 Service Authentication](SRS-019%20Service%20Authentication.ru.md), [SRS-018 Secrets Management](SRS-018%20Secrets%20Management.ru.md)

## Определение

JWT (JSON Web Token) Service Tokens — подписанные токены для аутентификации межсервисных запросов. Токен выдаётся вызывающим сервисом, подписывается приватным ключом и проверяется принимающим сервисом с помощью публичного ключа — без обращения к внешнему auth-сервису.

## Когда использовать

| Сценарий | Рекомендация |
|----------|-------------|
| Аутентификация внешних API | ✅ Предпочтительно |
| Межсервисное взаимодействие с общей PKI | ✅ Хороший выбор |
| Нужна stateless-проверка | ✅ Хороший выбор |
| Нужен немедленный отзыв токенов | ❌ Использовать OAuth 2.0 |
| Intra-cluster (единый домен доверия) | ❌ Использовать mTLS |

## Структура токена

```
Header.Payload.Signature
```

### Обязательные claims

```json
{
  "iss": "service-a",          // Issuer — имя вызывающего сервиса
  "sub": "service-a",          // Subject — то же что iss для service tokens
  "aud": "target-service",     // Audience — получатель
  "exp": 1710000000,           // Время истечения (Unix timestamp)
  "iat": 1709996400,           // Время выдачи
  "jti": "uuid-v4-unique-id",  // JWT ID — для защиты от replay-атак
  "scope": ["read", "write"]   // Разрешения
}
```

## Реализация

### Создание service token

```python
import jwt
import time
import uuid

def create_service_token(service_name: str, target_service: str,
                         private_key: str, scopes: list[str]) -> str:
    payload = {
        "iss": service_name,
        "sub": service_name,
        "aud": target_service,
        "exp": time.time() + 3600,   # 1 час
        "iat": time.time(),
        "jti": str(uuid.uuid4()),
        "scope": scopes
    }
    return jwt.encode(payload, private_key, algorithm="RS256")
```

### Проверка service token

```python
def verify_service_token(token: str, public_key: str,
                         expected_audience: str,
                         allowed_issuers: list[str]) -> dict:
    try:
        payload = jwt.decode(
            token,
            public_key,
            algorithms=["RS256"],
            audience=expected_audience,
            options={"require": ["exp", "iss", "aud", "jti", "scope"]}
        )

        if payload["iss"] not in allowed_issuers:
            raise ValueError(f"Issuer {payload['iss']} не в списке разрешённых")

        return payload

    except jwt.ExpiredSignatureError:
        raise AuthError("Токен истёк")
    except jwt.InvalidIssuerError:
        raise AuthError("Неверный issuer")
    except jwt.InvalidTokenError as e:
        raise AuthError(f"Невалидный токен: {e}")
```

### FastAPI Middleware

```python
from fastapi import FastAPI, Depends, HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

app = FastAPI()
security = HTTPBearer()

PUBLIC_KEY = open("/etc/keys/public.pem").read()
ALLOWED_SERVICES = {"service-a", "service-b", "service-c"}

def get_service(
    credentials: HTTPAuthorizationCredentials = Security(security)
) -> dict:
    payload = verify_service_token(
        token=credentials.credentials,
        public_key=PUBLIC_KEY,
        expected_audience="target-service",
        allowed_issuers=list(ALLOWED_SERVICES)
    )
    return payload

@app.get("/api/data")
async def get_data(service: dict = Depends(get_service)):
    return {"message": f"Hello from {service['iss']}"}
```

### Централизованный auth middleware

```python
@app.middleware("http")
async def auth_middleware(request: Request, call_next):
    if request.url.path in ["/health", "/live", "/ready"]:
        return await call_next(request)

    auth_header = request.headers.get("Authorization", "")
    if not auth_header.startswith("Bearer "):
        return JSONResponse(status_code=401,
                            content={"error": "Отсутствует заголовок авторизации"})

    try:
        payload = verify_service_token(
            token=auth_header.removeprefix("Bearer "),
            public_key=PUBLIC_KEY,
            expected_audience="this-service",
            allowed_issuers=ALLOWED_SERVICES
        )
        request.state.service_name = payload["iss"]
        request.state.scopes = payload.get("scope", [])
    except AuthError as e:
        return JSONResponse(status_code=401, content={"error": str(e)})

    return await call_next(request)
```

## Кэширование токенов

Чтобы не создавать новый токен при каждом запросе, кэшируйте токены до истечения срока:

```python
import threading
from dataclasses import dataclass

@dataclass
class CachedToken:
    token: str
    expires_at: float

_token_cache: dict[str, CachedToken] = {}
_lock = threading.Lock()

def get_service_token(target_service: str) -> str:
    with _lock:
        cached = _token_cache.get(target_service)
        # Обновляем за 60 секунд до истечения
        if cached and cached.expires_at > time.time() + 60:
            return cached.token

        token = create_service_token(
            service_name=MY_SERVICE_NAME,
            target_service=target_service,
            private_key=PRIVATE_KEY,
            scopes=["read"]
        )
        _token_cache[target_service] = CachedToken(
            token=token,
            expires_at=time.time() + 3600
        )
        return token
```

## Конфигурация

```
SERVICE_AUTH_JWT_ALGORITHM=RS256
SERVICE_AUTH_JWT_ISSUER=my-service
SERVICE_AUTH_JWT_AUDIENCE=target-service
SERVICE_AUTH_JWT_EXPIRATION=3600
SERVICE_AUTH_JWT_CLOCK_SKEW=60
SERVICE_AUTH_JWT_PRIVATE_KEY_PATH=/etc/keys/private.pem
SERVICE_AUTH_JWT_PUBLIC_KEY_PATH=/etc/keys/public.pem
```

## Мониторинг

```
service_jwt_issued_total (counter)
service_jwt_verification_total{result="success|failure"} (counter)
service_jwt_verification_duration_seconds (histogram)
service_jwt_token_cache_hits_total (counter)
service_jwt_token_cache_misses_total (counter)
```

## Best Practices

✅ **Делать**
* Использовать RS256 (асимметричный) — никогда не HS256 для service tokens
* Устанавливать короткое время истечения (максимум 1 час)
* Включать claim `jti` для защиты от replay-атак
* Кэшировать токены на клиентской стороне до момента близкого к истечению
* Регулярно ротировать ключевые пары (каждые 90 дней)
* Логировать все ошибки проверки с метаданными токена (не сам токен)

❌ **Не делать**
* Использовать симметричный HMAC (HS256) — требует раздачи общего секрета
* Устанавливать срок истечения более 24 часов
* Логировать или раскрывать значения токенов
* Доверять токенам без проверки подписи
* Пропускать валидацию `aud` (audience)

## Плюсы и минусы

**Плюсы:**
* Stateless-проверка — не нужен запрос к auth-сервису
* Простота использования — стандартный заголовок `Authorization: Bearer`
* Встроенное время истечения
* Поддерживается практически любым фреймворком

**Минусы:**
* Невозможность немедленного отзыва без blacklist
* Требует синхронизации времени (NTP)
* Размер токена ~1–2КБ на каждый запрос

## Дополнительные ресурсы

* [JWT RFC 7519](https://tools.ietf.org/html/rfc7519)
* [JWK RFC 7517](https://tools.ietf.org/html/rfc7517)
* [jwt.io debugger](https://jwt.io/)
* [PyJWT library](https://pyjwt.readthedocs.io/)
