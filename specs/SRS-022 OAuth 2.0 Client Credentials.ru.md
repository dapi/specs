# SRS-022 OAuth 2.0 Client Credentials (OAuth 2.0 для сервисов)

## Определение

OAuth 2.0 Client Credentials — grant flow (RFC 6749 §4.4), при котором сервис аутентифицируется с помощью `client_id` и `client_secret` для получения access token от централизованного authorization server. Токен используется для последующих API-вызовов.

## Когда использовать

| Сценарий | Рекомендация |
|----------|-------------|
| Нужно централизованное управление токенами | ✅ Предпочтительно |
| Нужен отзыв токенов | ✅ Предпочтительно |
| Много сервисов, один auth-сервер | ✅ Хороший выбор |
| Контроль доступа на основе scopes | ✅ Хороший выбор |
| Нет внешнего auth-сервера | ❌ Использовать JWT |
| Простые внутренние сервисы | ❌ Избыточно |

## Flow

```
Service A                    Auth Server              Service B
    │                            │                        │
    │── POST /oauth/token ───────>│                        │
    │   client_id + secret        │                        │
    │<── access_token ────────────│                        │
    │                            │                        │
    │── GET /api/data ────────────────────────────────────>│
    │   Authorization: Bearer <token>                      │
    │                            │                        │
    │                            │<── introspect token ───│
    │                            │─── token valid ────────>│
    │<── response ────────────────────────────────────────│
```

## Получение токена

```bash
curl -X POST https://auth.example.com/oauth/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=service-a" \
  -d "client_secret=secret-key" \
  -d "scope=users:read orders:read"

# Ответ
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "users:read orders:read"
}
```

## Реализация

### Token Client с кэшированием

```python
import time
import threading
import httpx
from dataclasses import dataclass
from pathlib import Path

@dataclass
class TokenInfo:
    access_token: str
    expires_at: float
    scope: str

class OAuth2Client:
    def __init__(self, token_url: str, client_id: str, client_secret: str):
        self._token_url = token_url
        self._client_id = client_id
        self._client_secret = client_secret
        self._tokens: dict[str, TokenInfo] = {}
        self._lock = threading.Lock()

    def _scope_cache_key(self, scope: str) -> str:
        # Нормализуем порядок scopes, чтобы эквивалентные запросы делили один cache key.
        return " ".join(sorted(scope.split()))

    def get_token(self, scope: str = "") -> str:
        with self._lock:
            cache_key = self._scope_cache_key(scope)
            # Обновляем за 60 секунд до истечения
            cached = self._tokens.get(cache_key)
            if cached and cached.expires_at > time.time() + 60:
                return cached.access_token

            token_info = self._fetch_token(scope)
            self._tokens[cache_key] = token_info
            return token_info.access_token

    def _fetch_token(self, scope: str) -> TokenInfo:
        response = httpx.post(
            self._token_url,
            data={
                "grant_type": "client_credentials",
                "client_id": self._client_id,
                "client_secret": self._client_secret,
                "scope": scope,
            },
        )
        response.raise_for_status()
        data = response.json()

        return TokenInfo(
            access_token=data["access_token"],
            expires_at=time.time() + data["expires_in"],
            scope=data.get("scope", ""),
        )

def load_secret(path: str) -> str:
    return Path(path).read_text().strip()

# Использование
auth_client = OAuth2Client(
    token_url="https://auth.example.com/oauth/token",
    client_id="service-a",
    client_secret=load_secret("/etc/secrets/service_client_secret"),
)

def call_service_b():
    token = auth_client.get_token(scope="users:read")
    response = httpx.get(
        "https://service-b/api/users",
        headers={"Authorization": f"Bearer {token}"}
    )
    return response.json()
```

### Token Introspection (серверная сторона)

```python
async def introspect_token(token: str) -> dict:
    """Проверка токена через endpoint интроспекции auth-сервера"""
    response = await httpx.AsyncClient().post(
        "https://auth.example.com/oauth/introspect",
        data={"token": token},
        auth=(RESOURCE_SERVER_ID, RESOURCE_SERVER_SECRET),
    )
    response.raise_for_status()
    data = response.json()

    if not data.get("active"):
        raise AuthError("Токен не активен")

    return data

# FastAPI dependency
async def require_token(
    credentials: HTTPAuthorizationCredentials = Security(HTTPBearer())
) -> dict:
    return await introspect_token(credentials.credentials)
```

### Локальная JWT-верификация (без интроспекции)

Если auth-сервер выдаёт JWT access tokens, проверку можно делать локально:

```python
import jwt

jwks_client = jwt.PyJWKClient("https://auth.example.com/.well-known/jwks.json")

async def verify_access_token(token: str) -> dict:
    """Проверка JWT access token с выбором signing key по kid"""
    signing_key = jwks_client.get_signing_key_from_jwt(token)

    payload = jwt.decode(
        token,
        signing_key.key,
        algorithms=["RS256"],
        audience="https://api.example.com",
        options={"require": ["exp", "iss", "aud"]}
    )
    return payload
```

## Конфигурация

```
SERVICE_AUTH_OAUTH2_TOKEN_URL=https://auth.example.com/oauth/token
SERVICE_AUTH_OAUTH2_CLIENT_ID=service-name
SERVICE_AUTH_OAUTH2_CLIENT_SECRET_PATH=/etc/secrets/service_client_secret
SERVICE_AUTH_OAUTH2_SCOPES=read,write
SERVICE_AUTH_OAUTH2_CACHE_TOKENS=true
SERVICE_AUTH_OAUTH2_TOKEN_CACHE_TTL=3500           # Чуть меньше expires_in
SERVICE_AUTH_OAUTH2_INTROSPECT_URL=https://auth.example.com/oauth/introspect
```

## Мониторинг

```
service_oauth2_token_requests_total{result="success|failure"} (counter)
service_oauth2_token_request_duration_seconds (histogram)
service_oauth2_token_cache_hits_total (counter)
service_oauth2_token_cache_misses_total (counter)
service_oauth2_introspection_total{result="active|inactive"} (counter)
service_oauth2_introspection_duration_seconds (histogram)
```

## Best Practices

✅ **Делать**
* Кэшировать токены до 60 секунд до истечения
* Загружать `client_secret` из Vault, KMS или из примонтированного secret-файла
* Использовать scopes для ограничения доступа до минимально необходимого
* Предпочитать локальную JWT-верификацию интроспекции — для производительности
* Устанавливать короткий `expires_in` (максимум 1 час)
* Регулярно ротировать `client_secret`

❌ **Не делать**
* Хранить `client_secret` в коде, в закоммиченных конфигах или в незашифрованных файлах
* Запрашивать токен при каждом API-вызове — всегда кэшировать
* Использовать слишком широкие scopes
* Пропускать валидацию срока истечения токена

## Плюсы и минусы

**Плюсы:**
* Стандартный протокол (RFC 6749) — широкая поддержка
* Централизованное управление токенами и отзыв
* Контроль доступа на основе scopes
* Можно ротировать `client_secret` без перезапуска сервисов (с кэшированием)

**Минусы:**
* Дополнительный HTTP-запрос для получения токена (нивелируется кэшированием)
* Требует инфраструктуры auth-сервера
* Высокая задержка при использовании интроспекции на каждый запрос

## Дополнительные ресурсы

* [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
* [Token Introspection RFC 7662](https://tools.ietf.org/html/rfc7662)
* [OAuth 2.0 for Machine-to-Machine](https://auth0.com/docs/get-started/authentication-and-authorization-flow/client-credentials-flow)
