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

Issuer формирует payload с полями `iss`, `sub`, `aud`, `exp`, `iat`, `jti` и опционально `scope`, затем подписывает его алгоритмом RS256 с использованием приватного ключа сервиса. Значение `jti` должно быть UUID v4 или аналогичным глобально уникальным идентификатором.

### Проверка service token (режим Static Public Key)

Receiver декодирует и проверяет токен с помощью публичного ключа issuer-а:

1. Проверить подпись RS256 с помощью сконфигурированного публичного ключа.
2. Отклонить, если `exp` в прошлом.
3. Отклонить, если `iat` в будущем дальше `SERVICE_AUTH_JWT_CLOCK_SKEW`.
4. Отклонить, если `now - iat > SERVICE_AUTH_JWT_EXPIRATION + SERVICE_AUTH_JWT_CLOCK_SKEW` (проверка возраста независимо от `exp`).
5. Отклонить, если `jti` отсутствует или пустой.
6. Отклонить, если `iss` не входит в список разрешённых issuer-ов.
7. Если включена replay-защита: отклонить, если `jti` уже встречался; иначе пометить как использованный с TTL = `exp - now`.

Если требуется защита от replay, хранилище использованных `jti` должно быть общим (например, Redis с семантикой `SETNX`) с TTL до `exp`. Наличие одного только `jti` не предотвращает повторное использование токена.

### JWKS Flow

```
Issuer                     Receiver                  Issuer JWKS
  |                            |                          |
  |--- POST /api (JWT) ------->|                          |
  |                            |-- read iss from header   |
  |                            |                          |
  |                            |  [cache miss]            |
  |                            |--- GET /.well-known/ --->|
  |                            |       jwks.json          |
  |                            |<-- { keys: [...] } ------|
  |                            |-- store in memory cache  |
  |                            |   (TTL = 5 min)          |
  |                            |                          |
  |                            |  [cache hit]             |
  |                            |-- read from cache        |
  |                            |                          |
  |                            |-- verify signature       |
  |                            |-- validate claims        |
  |<-- 200 OK / 401 Unauth. ---|                          |
```

### Проверка через JWKS

В JWKS-режиме receiver получает публичные ключи динамически от issuer-а, не храня статический файл ключа. Ключи кэшируются в памяти на 5 минут (фиксированный TTL, не конфигурируется).

Шаги проверки:

1. Извлечь `iss` из токена (без проверки подписи).
2. Отклонить, если `iss` не входит в `SERVICE_AUTH_JWT_JWKS_CLIENTS`.
3. Получить JWKS по URL из `SERVICE_AUTH_JWT_JWKS_URL_TEMPLATE` (подставив `{client}` = `iss`), или вернуть кэшированный результат, если он ещё свежий.
4. Если в заголовке токена задан `kid` — выбрать только ключ с совпадающим `kid`; иначе перебрать все ключи набора.
5. Попытаться проверить подпись каждым ключом-кандидатом по очереди. Остановиться на первом успешном.
6. Применить те же проверки `iat`/`exp`/`jti`/replay, что и в режиме Static Public Key.
7. Вернуть верифицированный payload или 401, если ни один ключ не подошёл.

JWKS может содержать несколько ключей для поддержки ротации. Если `kid` задан — используется только совпадающий ключ; иначе перебираются все.

### JWKS-эндпоинт (требование к issuer)

В JWKS-режиме issuer **обязан** предоставлять `GET /.well-known/jwks.json`, возвращающий действующие публичные ключи в формате JWK Set (RFC 7517). При ротации ключей старый ключ должен оставаться в JWKS до истечения всех выданных им токенов — то есть не менее `SERVICE_AUTH_JWT_EXPIRATION` секунд после смены.

### Кэширование токенов (на стороне issuer)

Чтобы не создавать новый токен при каждом исходящем запросе, issuer должен кэшировать токены и переиспользовать их до момента близкого к истечению (например, обновлять за 60 секунд до `exp`). Кэш ключируется по имени целевого сервиса и хранится в памяти процесса.

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

## Рекомендуемые библиотеки

### Python

| Роль | Библиотека | Примечание |
|------|-----------|-----------|
| Issuer + Receiver | `PyJWT` + `cryptography` | RS256, поддержка JWK через `jwt.algorithms.RSAAlgorithm.from_jwk()` |
| Receiver (JWKS) | `python-jose` | Встроенная поддержка JWKS fetch и кэширования |

### Node.js

| Роль | Библиотека | Примечание |
|------|-----------|-----------|
| Issuer + Receiver | `jsonwebtoken` | Стандарт де-факто, RS256, без встроенного JWKS |
| Receiver (JWKS) | `jwks-rsa` | JWKS fetch и кэш, интегрируется с `jsonwebtoken` |
| Receiver (JWKS) | `jose` | Полная поддержка JWKS, современный API, ESM/CJS |

### Go

| Роль | Библиотека | Примечание |
|------|-----------|-----------|
| Issuer + Receiver | `golang-jwt/jwt` | Стандарт для Go, RS256 |
| Receiver (JWKS) | `MicahParks/keyfunc` | JWKS fetch и кэш, интегрируется с `golang-jwt/jwt` |

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
