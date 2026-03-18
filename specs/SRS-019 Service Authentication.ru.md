# SRS-019 Service Authentication (Аутентификация сервисов)

## Определение

Service Authentication — это процесс проверки подлинности сервиса, который делает запрос, для предотвращения несанкционированного доступа к ресурсам.

## Типы аутентификации

| Тип | Спецификация | Лучший для |
|-----|-------------|------------|
| mTLS | [SRS-020](SRS-020%20mTLS%20Authentication.ru.md) | Service-to-service внутри кластера |
| JWT Service Tokens | [SRS-021](SRS-021%20JWT%20Service%20Tokens.ru.md) | Внешние API, межсервисное взаимодействие |
| OAuth 2.0 Client Credentials | [SRS-022](SRS-022%20OAuth%202.0%20Client%20Credentials.ru.md) | Централизованный auth с управлением токенами |
| API Keys | [SRS-023](SRS-023%20API%20Keys%20Authentication.ru.md) | Простые интеграции, legacy-системы |

## Руководство по выбору

| Сценарий | Рекомендация |
|----------|-------------|
| Service-to-service внутри кластера | [mTLS](SRS-020%20mTLS%20Authentication.ru.md) |
| Внешние сервисы | [JWT](SRS-021%20JWT%20Service%20Tokens.ru.md) или [OAuth 2.0](SRS-022%20OAuth%202.0%20Client%20Credentials.ru.md) |
| Простые интеграции | [API Keys](SRS-023%20API%20Keys%20Authentication.ru.md) |
| Высокие требования к безопасности | [mTLS](SRS-020%20mTLS%20Authentication.ru.md) + [JWT](SRS-021%20JWT%20Service%20Tokens.ru.md) |
| Legacy-системы | [API Keys](SRS-023%20API%20Keys%20Authentication.ru.md) |

## Общие требования

Все типы аутентификации ДОЛЖНЫ:
* Проверять идентификацию до предоставления доступа
* Логировать все события аутентификации
* Использовать timing-safe сравнение секретов
* Регулярно ротировать учётные данные
* Никогда не хранить секреты в коде

## Мониторинг

```
service_auth_requests_total (counter, labels: type=mtls|jwt|oauth2|apikey)
service_auth_failures_total (counter, labels: type, reason)
service_auth_duration_seconds (histogram, labels: type)
```

## Дополнительные ресурсы

* [SRS-020 mTLS Authentication](SRS-020%20mTLS%20Authentication.ru.md)
* [SRS-021 JWT Service Tokens](SRS-021%20JWT%20Service%20Tokens.ru.md)
* [SRS-022 OAuth 2.0 Client Credentials](SRS-022%20OAuth%202.0%20Client%20Credentials.ru.md)
* [SRS-023 API Keys Authentication](SRS-023%20API%20Keys%20Authentication.ru.md)
