# SRS-021 JWT Service Tokens (JWT-токены для сервисов)

## Определение

JWT (JSON Web Token) Service Tokens — подписанные токены для аутентификации межсервисных запросов. Токен выдаётся вызывающим сервисом, подписывается приватным ключом и проверяется принимающим сервисом с помощью публичного ключа — без обращения к внешнему auth-сервису.

## Когда использовать

| Сценарий | Рекомендация |
|----------|-------------|
| Аутентификация внешних API | ✅ Предпочтительно |
| Межсервисное взаимодействие с общей PKI | ✅ Хороший выбор |
| Нужна stateless-проверка | ✅ Хороший выбор |
| Несколько issuer-ов, один receiver | ✅ JWKS-режим |
| Нужен немедленный отзыв токенов | ❌ Использовать OAuth 2.0 |
| Intra-cluster (единый домен доверия) | ❌ Использовать mTLS |

## Структура токена

```
Header.Payload.Signature
```

### Обязательные claims

```json
{
  "exp": 1710000000,           // Время истечения (Unix timestamp) — всегда обязателен
  "iat": 1709996400,           // Время выдачи — всегда обязателен
  "jti": "uuid-v4-unique-id",  // JWT ID — уникальный идентификатор для аудита/поиска replay, всегда обязателен
  "iss": "service-a",          // Issuer — обязателен если задан SERVICE_AUTH_JWT_ISSUER
  "aud": "target-service",     // Audience — обязателен если задан SERVICE_AUTH_JWT_AUDIENCE
  "sub": "service-a",          // Subject — рекомендован
  "scope": ["read", "write"]   // Разрешения — опционально
}
```

**Правила валидации:**

| Claim | Обязательность | Поведение |
|-------|---------------|-----------|
| `exp` | Всегда | 401 если истёк |
| `iat` | Всегда | 401 если отсутствует; 401 если токен выдан в будущем дальше `SERVICE_AUTH_JWT_CLOCK_SKEW`; 401 если `now - iat > SERVICE_AUTH_JWT_EXPIRATION + SERVICE_AUTH_JWT_CLOCK_SKEW` |
| `jti` | Всегда | 401 если отсутствует или пустая строка; если включена replay-защита, 401 при повторном `jti` |
| `iss` | Если задан `SERVICE_AUTH_JWT_ISSUER` | 401 если значение не совпадает с `SERVICE_AUTH_JWT_ISSUER` |
| `aud` | Если задан `SERVICE_AUTH_JWT_AUDIENCE` (рекомендуется в production) | 401 если значение не содержит `SERVICE_AUTH_JWT_AUDIENCE` |
| `kid` | Опционально | Используется для выбора совпадающего ключа из JWKS; если не задан, перебираются все ключи набора |
| `scope` | Опционально | Используется для проверки разрешений если присутствует |

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
        "exp": int(time.time()) + 3600,   # 1 час
        "iat": int(time.time()),
        "jti": str(uuid.uuid4()),
        "scope": scopes
    }
    return jwt.encode(payload, private_key, algorithm="RS256")
```

### Проверка service token

```python
def verify_service_token(token: str, public_key: str,
                         expected_audience: str | None,
                         allowed_issuers: set[str],
                         jwt_expiration: int,
                         jwt_clock_skew: int = 60,
                         replay_cache=None) -> dict:
    try:
        payload = jwt.decode(
            token,
            public_key,
            algorithms=["RS256"],
            audience=expected_audience if expected_audience else None,
            options={
                "require": ["exp", "iat", "jti"],
                "verify_aud": bool(expected_audience),
            }
        )

        now = int(time.time())
        if payload["iat"] > now + jwt_clock_skew:
            raise AuthError("Токен выдан в будущем")

        # Проверка возраста по iat — отклоняем даже если exp ещё не истёк
        age = now - payload["iat"]
        if age > jwt_expiration + jwt_clock_skew:
            raise AuthError("Токен слишком старый")

        jti = payload.get("jti")
        if not jti:
            raise AuthError("Отсутствует jti")

        if allowed_issuers and payload.get("iss") not in allowed_issuers:
            raise AuthError(f"Issuer {payload.get('iss')} не разрешён")

        if replay_cache is not None:
            ttl = max(1, int(payload["exp"] - now))
            if not replay_cache.mark_first_seen(jti, ttl=ttl):
                raise AuthError("Обнаружен replay")

        return payload

    except jwt.ExpiredSignatureError:
        raise AuthError("Токен истёк")
    except jwt.InvalidIssuerError:
        raise AuthError("Неверный issuer")
    except jwt.InvalidTokenError as e:
        raise AuthError(f"Невалидный токен: {e}")
```

Если требуется защита от replay, `replay_cache` должен быть реализован поверх общего хранилища вроде Redis с семантикой `SETNX` и TTL до `exp`. Наличие одного только `jti` не предотвращает повторное использование токена.

### Проверка через JWKS

В JWKS-режиме receiver получает публичные ключи динамически от issuer-а, не храня статический файл ключа. Ключи кэшируются в памяти на 5 минут.

```python
import time
import threading
import urllib.request
import json

_jwks_cache: dict[str, tuple[list, float]] = {}  # client -> (keys, fetched_at)
_jwks_lock = threading.Lock()
JWKS_CACHE_TTL = 300  # 5 минут

def _fetch_jwks(client: str, url_template: str) -> list:
    url = url_template.replace("{client}", client)
    with urllib.request.urlopen(url, timeout=5) as resp:
        return json.loads(resp.read())["keys"]

def get_jwks_keys(client: str, url_template: str) -> list:
    with _jwks_lock:
        cached = _jwks_cache.get(client)
        if cached and time.time() - cached[1] < JWKS_CACHE_TTL:
            return cached[0]
        keys = _fetch_jwks(client, url_template)
        _jwks_cache[client] = (keys, time.time())
        return keys

def verify_service_token_jwks(token: str,
                               allowed_clients: set[str],
                               url_template: str,
                               expected_audience: str | None,
                               jwt_expiration: int,
                               jwt_clock_skew: int = 60,
                               replay_cache=None) -> dict:
    header = jwt.get_unverified_header(token)
    issuer = jwt.decode(token, options={"verify_signature": False}).get("iss")

    if issuer not in allowed_clients:
        raise AuthError(f"Issuer {issuer} не в списке разрешённых клиентов")

    keys = get_jwks_keys(issuer, url_template)
    kid = header.get("kid")

    # Фильтруем по kid если задан, иначе перебираем все ключи
    candidates = [k for k in keys if not kid or k.get("kid") == kid]
    if not candidates:
        raise AuthError(f"Не найден ключ для kid={kid}")

    last_error = None
    for jwk in candidates:
        try:
            public_key = jwt.algorithms.RSAAlgorithm.from_jwk(jwk)
            payload = jwt.decode(
                token,
                public_key,
                algorithms=["RS256"],
                audience=expected_audience if expected_audience else None,
                options={
                    "require": ["exp", "iat", "jti"],
                    "verify_aud": bool(expected_audience),
                }
            )
            # Те же проверки iat/jti/replay что и в verify_service_token
            now = int(time.time())
            if payload["iat"] > now + jwt_clock_skew:
                raise AuthError("Токен выдан в будущем")
            if now - payload["iat"] > jwt_expiration + jwt_clock_skew:
                raise AuthError("Токен слишком старый")
            jti = payload.get("jti")
            if not jti:
                raise AuthError("Отсутствует jti")
            if replay_cache is not None:
                ttl = max(1, int(payload["exp"] - now))
                if not replay_cache.mark_first_seen(jti, ttl=ttl):
                    raise AuthError("Обнаружен replay")
            return payload
        except (jwt.InvalidTokenError, AuthError) as e:
            last_error = e

    raise AuthError(f"Проверка токена не прошла: {last_error}")
```

JWKS может содержать несколько ключей (ротация). Если `kid` задан — используется только совпадающий ключ; если нет — перебираются все по очереди.

### JWKS-эндпоинт (требование к issuer)

В JWKS-режиме issuer **обязан** предоставлять `GET /.well-known/jwks.json`, возвращающий действующие публичные ключи в формате JWK Set (RFC 7517). При ротации ключей старый ключ должен оставаться в JWKS до истечения всех выданных им токенов — то есть не менее `SERVICE_AUTH_JWT_EXPIRATION` секунд после смены.

Пример ответа:

```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "kid": "2024-01-key",
      "alg": "RS256",
      "n": "...",
      "e": "AQAB"
    }
  ]
}
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
        allowed_issuers=ALLOWED_SERVICES,
        jwt_expiration=3600,
        jwt_clock_skew=60,
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
            allowed_issuers=ALLOWED_SERVICES,
            jwt_expiration=3600,
            jwt_clock_skew=60,
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

**Issuer** (подписывает токены, хранит приватный ключ):

```
SERVICE_AUTH_JWT_ALGORITHM=RS256
SERVICE_AUTH_JWT_ISSUER=my-service
SERVICE_AUTH_JWT_EXPIRATION=3600
SERVICE_AUTH_JWT_PRIVATE_KEY_PATH=/etc/keys/private.pem
```

**Receiver — режим Static Public Key** (проверяет токены через локальный файл публичного ключа):

```
SERVICE_AUTH_JWT_ALGORITHM=RS256
SERVICE_AUTH_JWT_AUDIENCE=target-service
SERVICE_AUTH_JWT_CLOCK_SKEW=60
SERVICE_AUTH_JWT_PUBLIC_KEY_PATH=/etc/keys/public.pem
```

**Receiver — JWKS-режим** (получает публичные ключи динамически от issuer-а):

```
SERVICE_AUTH_JWT_ALGORITHM=RS256
SERVICE_AUTH_JWT_AUDIENCE=target-service
SERVICE_AUTH_JWT_CLOCK_SKEW=60
SERVICE_AUTH_JWT_JWKS_CLIENTS=service-a,service-b
SERVICE_AUTH_JWT_JWKS_URL_TEMPLATE=https://{client}/.well-known/jwks.json
```

Правила выбора режима:
- Если задан `PUBLIC_KEY_PATH` → режим Static Public Key
- Если заданы `JWKS_CLIENTS` и `JWKS_URL_TEMPLATE` → JWKS-режим
- Если заданы оба (или ни одного) → сервис должен завершить запуск с ошибкой конфигурации

## Мониторинг

```
service_jwt_issued_total (counter)
service_jwt_verification_total{result="success|failure"} (counter)
service_jwt_verification_duration_seconds (histogram)
service_jwt_token_cache_hits_total (counter)
service_jwt_token_cache_misses_total (counter)
service_jwt_jwks_fetch_total{client="...", result="success|failure"} (counter)
service_jwt_jwks_cache_hits_total{client="..."} (counter)
```

## Best Practices

✅ **Делать**
* Использовать RS256 (асимметричный) — никогда не HS256 для service tokens
* Устанавливать короткое время истечения (максимум 1 час)
* Включать claim `jti` и подключать общее хранилище, если нужна replay-защита
* Кэшировать токены на клиентской стороне до момента близкого к истечению
* Регулярно ротировать ключевые пары (каждые 90 дней)
* Логировать все ошибки проверки с метаданными токена (не сам токен)
* Отклонять токены с `iat` в будущем дальше допустимого clock skew
* В JWKS-режиме включать `kid` в заголовок JWT для эффективного выбора ключа
* Хранить в JWKS все активные ключи при ротации (overlap-период = время жизни токена)

❌ **Не делать**
* Использовать симметричный HMAC (HS256) — требует раздачи общего секрета
* Устанавливать срок истечения более 1 часа в production
* Логировать или раскрывать значения токенов
* Доверять токенам без проверки подписи
* Пропускать валидацию `aud` (audience) в production
* Удалять ключ из JWKS до истечения выданных им токенов

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
