# SRS-020 mTLS Authentication (Взаимная TLS-аутентификация)

## Определение

Mutual TLS (mTLS) — метод аутентификации сервисов, при котором и клиент, и сервер предоставляют X.509-сертификаты, подписанные доверенным Центром сертификации (CA), устанавливая взаимное криптографическое подтверждение личности.

## Когда использовать

| Сценарий | Рекомендация |
|----------|-------------|
| Service-to-service внутри кластера | ✅ Предпочтительно |
| Высокие требования к безопасности | ✅ Обязательно |
| Zero-trust сетевая модель | ✅ Обязательно |
| Внешние публичные API | ❌ Использовать JWT/OAuth |
| Legacy-системы | ❌ Слишком сложно |

## Как работает

```
Клиент                          Сервер
  │                               │
  │──── ClientHello ─────────────>│
  │<─── ServerHello + Cert ───────│
  │──── Client Cert ─────────────>│
  │     (оба проверяют друг друга) │
  │<═══════ TLS-сессия ═══════════│
```

1. Сервер предоставляет свой сертификат
2. Клиент проверяет сертификат сервера через CA
3. Клиент предоставляет свой сертификат
4. Сервер проверяет сертификат клиента через CA
5. Установлена взаимная TLS-сессия

## Конфигурация

### Nginx

```yaml
server {
    listen 443 ssl;

    ssl_certificate /etc/ssl/certs/server.crt;
    ssl_certificate_key /etc/ssl/private/server.key;

    # CA для проверки клиентских сертификатов
    ssl_client_certificate /etc/ssl/certs/ca.crt;
    ssl_verify_client on;
    ssl_verify_depth 2;

    location / {
        proxy_set_header X-SSL-Client-Serial $ssl_client_serial;
        proxy_set_header X-SSL-Client-DN $ssl_client_s_dn;
        proxy_set_header X-SSL-Client-Verify $ssl_client_verify;
        proxy_pass http://app;
    }
}
```

Эти identity-заголовки ДОЛЖНЫ добавляться только доверенным TLS-terminating proxy или service mesh sidecar. Приложение НЕ ДОЛЖНО быть доступно напрямую из недоверенных сетей, а любые клиентские `X-SSL-Client-*` заголовки должны удаляться или перезаписываться на периметре.

### Kubernetes / Istio

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

### Переменные окружения

```
SERVICE_AUTH_MTLS_VERIFY=true
SERVICE_AUTH_MTLS_CA_CERT_PATH=/etc/ssl/certs/ca.crt
SERVICE_AUTH_MTLS_CERT_PATH=/etc/ssl/certs/client.crt
SERVICE_AUTH_MTLS_KEY_PATH=/etc/ssl/private/client.key
SERVICE_AUTH_MTLS_VERIFY_DEPTH=2
```

## Реализация

### Клиент (Python)

```python
import ssl
import httpx

def create_mtls_client(cert_path: str, key_path: str, ca_path: str) -> httpx.Client:
    ssl_context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
    ssl_context.load_cert_chain(cert_path, key_path)
    ssl_context.load_verify_locations(ca_path)
    ssl_context.verify_mode = ssl.CERT_REQUIRED
    return httpx.Client(verify=ssl_context)

client = create_mtls_client(
    cert_path="/etc/ssl/certs/client.crt",
    key_path="/etc/ssl/private/client.key",
    ca_path="/etc/ssl/certs/ca.crt"
)

response = client.get("https://target-service/api/data")
```

### Серверная проверка сертификата (FastAPI)

```python
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()

@app.middleware("http")
async def verify_mtls(request: Request, call_next):
    # Доверять этим заголовкам можно только от локального/доверенного proxy,
    # который завершил mTLS. Прямой доступ к порту приложения должен быть закрыт.
    client_verify = request.headers.get("X-SSL-Client-Verify")
    if client_verify != "SUCCESS":
        raise HTTPException(status_code=401, detail="Client certificate required")

    # Извлекаем имя сервиса из DN сертификата: CN=service-name,O=company
    client_dn = request.headers.get("X-SSL-Client-DN", "")
    request.state.service_name = extract_cn_from_dn(client_dn)

    return await call_next(request)

def extract_cn_from_dn(dn: str) -> str:
    for part in dn.split(","):
        if part.strip().startswith("CN="):
            return part.strip()[3:]
    return ""
```

## Управление сертификатами

### Жизненный цикл сертификата

```
Генерация CA → Выпуск серверного сертификата → Выпуск клиентского → Деплой → Мониторинг срока → Ротация
```

### Рекомендованный срок действия

| Среда | Срок действия | Триггер ротации |
|-------|--------------|-----------------|
| Development | 1 год | Вручную |
| Staging | 90 дней | При 75% срока |
| Production | 30 дней | При 75% срока |
| Высокая безопасность | 24 часа | Автоматически (SPIRE) |

### Мониторинг срока истечения

```bash
# Проверить срок действия сертификата
openssl x509 -in /etc/ssl/certs/client.crt -noout -enddate

# Пороги оповещений: 30д, 7д, 1д до истечения
```

### Автоматическая ротация через SPIFFE/SPIRE

```yaml
agent:
  data_dir: /opt/spire/data/agent
  log_level: INFO
  server_address: spire-server
  server_port: 8081
  trust_domain: example.org
```

## Мониторинг

```
service_mtls_handshake_total{result="success|failure"} (counter)
service_mtls_handshake_duration_seconds (histogram)
service_mtls_cert_expiry_seconds{service="name"} (gauge)
service_mtls_verification_failures_total (counter)
```

## Best Practices

✅ **Делать**
* Использовать mTLS для всей intra-cluster коммуникации
* Автоматизировать ротацию сертификатов (SPIFFE/SPIRE, cert-manager)
* Настроить оповещения об истечении за 30/7/1 день
* Использовать краткоживущие сертификаты (24ч–30д) с автообновлением
* Хранить приватные ключи в защищённом хранилище (Vault, KMS)
* Доверять forwarded certificate headers только от защищённого proxy или sidecar

❌ **Не делать**
* Использовать самоподписанные сертификаты в production без CA
* Хранить приватные ключи в переменных окружения или конфиг-файлах
* Устанавливать срок действия сертификата более 1 года
* Пропускать проверку клиентских сертификатов

## Плюсы и минусы

**Плюсы:**
* Наивысший уровень безопасности — криптографическая идентификация, невозможность подделки
* Нет общих секретов — у каждого сервиса своя пара ключей
* Взаимная проверка — оба участника аутентифицируют друг друга
* Работает на транспортном уровне — прозрачно для прикладного кода

**Минусы:**
* Сложность управления сертификатами
* Накладные расходы на TLS handshake (~1–5мс для новых соединений)
* Требует PKI-инфраструктуры
* Ротация сертификатов требует координации

## Дополнительные ресурсы

* [SPIFFE/SPIRE](https://spiffe.io/)
* [cert-manager для Kubernetes](https://cert-manager.io/)
* [Istio mTLS](https://istio.io/docs/concepts/security/)
* [RFC 5246 — TLS Protocol](https://tools.ietf.org/html/rfc5246)
