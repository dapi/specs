# SRG-006 Audit Logging (Аудитные логи)

Audit Logging - это практика систематической записи аудитных событий, связанных с безопасностью, доступом и важными операциями в системе для последующего анализа и расследования инцидентов.

---

## Цели аудитного логирования

### Безопасность
- Обнаружение несанкционированного доступа
- Расследование инцидентов безопасности
- Соблюдение compliance требований (GDPR, HIPAA, PCI DSS, SOX)
- Форензик анализ

### Операционная прозрачность
- Отслеживание важных операций
- Восстановление цепочки действий
- Анализ причинно-следственных связей
- Доказательство выполнения операций

### Compliance
- Сохранение истории по требованию регуляторов
- Аудиты безопасности
- Хранение данных в соответствии с законодательством

---

## Что нужно логировать (Audit Events)

### Аутентификация и авторизация
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "event_type": "user.login",
  "user_id": "user_12345",
  "ip_address": "192.168.1.100",
  "user_agent": "Mozilla/5.0...",
  "result": "success",
  "session_id": "sess_abc123",
  "auth_method": "password"
}
```

Важные события:
- `user.login` - успешный/неуспешный вход
- `user.logout` - выход из системы
- `session.created` - создание сессии
- `session.destroyed` - уничтожение сессии
- `privilege.escalation` - повышение привилегий
- `access.denied` - отказ в доступе
- `auth.failed` - неудачная аутентификация

### Доступ к данным

```json
{
  "timestamp": "2024-01-15T10:35:00Z",
  "event_type": "data.accessed",
  "user_id": "user_12345",
  "resource_type": "user_record",
  "resource_id": "user_67890",
  "action": "read",
  "result": "success",
  "data_classification": "PII",
  "query": "SELECT * FROM users WHERE id = ?"
}
```

Важные события:
- `data.accessed` - доступ к данным
- `data.created` - создание записи
- `data.modified` - изменение данных
- `data.deleted` - удаление данных
- `data.exported` - экспорт данных
- `data.shared` - предоставление доступа

### Управление пользователями и правами

```json
{
  "timestamp": "2024-01-15T11:00:00Z",
  "event_type": "user.created",
  "admin_id": "admin_001",
  "target_user_id": "user_99999",
  "target_role": "editor",
  "permissions": ["read", "write"]
}
```

Важные события:
- `user.created` - создание пользователя
- `user.modified` - изменение данных пользователя
- `user.deleted` - удаление пользователя
- `role.assigned` - назначение роли
- `permission.granted` - предоставление прав
- `permission.revoked` - отзыв прав
- `group.membership.changed` - изменение в группах

### Управление конфигурациями

```json
{
  "timestamp": "2024-01-15T11:30:00Z",
  "event_type": "config.changed",
  "user_id": "admin_001",
  "resource": "payment_gateway",
  "changes": {
    "stripe_key": "*** CHANGED ***",
    "webhook_url": "https://api.new.example.com/webhook"
  },
  "previous_version": "config_v45",
  "new_version": "config_v46",
  "reason": "Migration to new payment provider"
}
```

### Финансовые операции

```json
{
  "timestamp": "2024-01-15T12:00:00Z",
  "event_type": "payment.processed",
  "user_id": "user_12345",
  "amount": 99.99,
  "currency": "USD",
  "payment_method": "card_6789",
  "transaction_id": "txn_abc123def456",
  "status": "completed",
  "compliance_check": "passed"
}
```

### Административные действия

```json
{
  "timestamp": "2024-01-15T14:00:00Z",
  "event_type": "admin.action",
  "admin_id": "super_admin",
  "action": "database_backup_created",
  "affected_systems": ["primary_db", "replica_db"],
  "duration_ms": 45000
}
```

---

## Формат аудитных логов

### Обязательные поля (Minimum Viable)

```json
{
  "timestamp": "2024-01-15T10:30:00Z",           # ISO 8601 с часовым поясом
  "event_type": "user.login",                    # Тип события
  "actor_id": "user_12345",                      # Кто выполнил действие
  "result": "success",                           # success | failure
  "correlation_id": "corr_abc123def456"          # Для трассировки
}
```

### Рекомендуемые поля (Standard)

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "event_type": "user.login",
  "actor": {
    "id": "user_12345",
    "type": "user",
    "ip_address": "192.168.1.100",
    "user_agent": "Mozilla/5.0..."
  },
  "action": "login",
  "resource": {
    "type": "user_account",
    "id": "account_67890"
  },
  "result": "success",
  "correlation_id": "corr_abc123def456",
  "session_id": "sess_xyz789",
  "auth_method": "password",
  "compliance_tags": ["GDPR", "SOX"]
}
```

### Полный формат (Comprehensive)

```json
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "event_type": "payment.processed",
  "version": "1.0",

  "actor": {
    "id": "user_12345",
    "type": "user",
    "identity_provider": "internal",
    "roles": ["customer", "premium"],
    "ip_address": "192.168.1.100",
    "user_agent": "Mozilla/5.0...",
    "location": {
      "country": "US",
      "city": "New York"
    }
  },

  "action": {
    "type": "payment.processed",
    "parameters": {
      "amount": 99.99,
      "currency": "USD",
      "payment_method": "card_6789"
    }
  },

  "resource": {
    "type": "transaction",
    "id": "txn_abc123def456",
    "owner_id": "user_12345",
    "data_classification": "PCI"
  },

  "result": {
    "status": "success",
    "code": 200,
    "message": "Payment processed successfully",
    "reason": null
  },

  "before_state": {
    "user_balance": 150.00
  },

  "after_state": {
    "user_balance": 50.01
  },

  "metadata": {
    "correlation_id": "corr_abc123def456",
    "session_id": "sess_xyz789",
    "request_id": "req_12345",
    "trace_id": "trace_abc123",
    "tags": ["production", "eu-west-1", "v2.1.0"]
  },

  "compliance": {
    "retention_period_days": 2555,
    "regulations": ["PCI-DSS", "GDPR"],
    "auditor_notes": "Standard purchase transaction"
  }
}
```

---

## Уровни логирования

### Level 0: Минимальный
- Вход/выход пользователей
- Неудачные аутентификации
- Изменения прав доступа

### Level 1: Стандартный (рекомендуется)
- Все события Level 0
- Критические операции с данными
- Финансовые транзакции
- Изменения конфигурации
- Административные действия

### Level 2: Детальный
- Все события Level 1
- Доступ к чувствительным данным (PII, PHI)
- Все изменения данных
- Попытки доступа (успешные и неуспешные)
- Все API вызовы с параметрами

### Level 3: Debug (для forensics)
- Все события Level 2
- Все состояния до и после
- Полные request/response (без секретов)
- Network activity
- System calls

---

## Интеграция в приложения

### Django Middleware

```python
class AuditLogMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        actor_id = request.user.id if request.user.is_authenticated else 'anonymous'

        # Логируем запрос
        logger.info({
            'event_type': 'request.received',
            'actor_id': actor_id,
            'method': request.method,
            'path': request.path,
            'ip_address': get_client_ip(request)
        })

        response = self.get_response(request)

        # Логируем ответ
        logger.info({
            'event_type': 'response.sent',
            'actor_id': actor_id,
            'status_code': response.status_code,
            'content_length': len(response.content)
        })

        return response
```

### Ruby on Rails

```ruby
class AuditLog
  def self.log(current_user, action, resource, result)
    AuditLogEntry.create!(
      timestamp: Time.now.utc,
      user_id: current_user&.id,
      user_ip: RequestContext.ip,
      action: action,
      resource_type: resource.class.name,
      resource_id: resource.id,
      result: result,
      session_id: RequestContext.session_id,
      correlation_id: RequestContext.correlation_id
    )
  end
end

# Использование
class PaymentsController < ApplicationController
  def create
    payment = Payment.new(payment_params)

    if payment.save
      AuditLog.log(current_user, 'payment.created', payment, 'success')
      render json: payment
    else
      AuditLog.log(current_user, 'payment.created', payment, 'failure')
      render json: {errors: payment.errors}, status: :unprocessable_entity
    end
  end
end
```

### Node.js (Express)

```javascript
const auditLogger = (req, res, next) => {
  const startTime = Date.now();
  const actorId = req.user ? req.user.id : 'anonymous';

  // Логируем вход
  logger.info({
    event_type: 'request.start',
    actor_id: actorId,
    method: req.method,
    path: req.path,
    ip_address: req.ip,
    user_agent: req.get('User-Agent')
  });

  // Перехватываем ответ
  const originalSend = res.send;
  res.send = function(body) {
    const duration = Date.now() - startTime;

    logger.info({
      event_type: 'response.complete',
      actor_id: actorId,
      method: req.method,
      path: req.path,
      status_code: res.statusCode,
      duration_ms: duration,
      response_size: body.length
    });

    return originalSend.call(this, body);
  };

  next();
};

app.use(auditLogger);
```

---

## Настройка хранения

### Elasticsearch

```python
# Индекс с lifecycle policy
PUT /_index_template/audit_logs
{
  "index_patterns": ["audit-logs-*"],
  "template": {
    "settings": {
      "index.lifecycle.name": "audit_logs_policy",
      "index.lifecycle.rollover_alias": "audit-logs"
    },
    "mappings": {
      "properties": {
        "timestamp": {"type": "date"},
        "event_type": {"type": "keyword"},
        "actor_id": {"type": "keyword"},
        "result": {"type": "keyword"},
        "ip_address": {"type": "ip"},
        "location": {"type": "geo_point"}
      }
    }
  }
}
```

### S3 для долгосрочного хранения

```python
import boto3
from datetime import datetime

class S3AuditStorage:
    def __init__(self, bucket):
        self.s3 = boto3.client('s3')
        self.bucket = bucket

    def store_log(self, audit_entry):
        date_str = datetime.utcnow().strftime('%Y/%m/%d')
        key = f"audit-logs/{date_str}/{audit_entry['correlation_id']}.json"

        self.s3.put_object(
            Bucket=self.bucket,
            Key=key,
            Body=json.dumps(audit_entry),
            ServerSideEncryption='AES256',
            StorageClass='GLACIER' if is_old(audit_entry) else 'STANDARD'
        )
```

---

## Анализ и мониторинг

### Dashboard в Kibana

```json
{
  "dashboard": {
    "title": "Audit Log Dashboard",
    "panels": [
      {
        "type": "timeseries",
        "title": "Аутентификации за 24 часа",
        "query": "event_type:user.login"
      },
      {
        "type": "metric",
        "title": "Неудачных входов",
        "query": "event_type:user.login AND result:failure"
      },
      {
        "type": "table",
        "title": "Критические события",
        "query": "compliance.regulations:*"
      }
    ]
  }
}
```

### Оповещения

```yaml
# Elasticsearch Watcher
PUT _watcher/watch/failed_logins
{
  "trigger": {
    "schedule": {"interval": "5m"}
  },
  "input": {
    "search": {
      "request": {
        "indices": ["audit-logs-*"],
        "body": {
          "query": {
            "bool": {
              "must": [
                {"term": {"event_type": "user.login"}},
                {"term": {"result": "failure"}}
              ],
              "filter": {
                "range": {
                  "timestamp": {"gte": "now-5m"}
                }
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {"ctx.payload.hits.total": {"gte": 10}}
  },
  "actions": {
    "send_email": {
      "email": {
        "to": "security@example.com",
        "subject": "⚠️ Multiple failed login attempts detected",
        "body": "More than 10 failed logins in last 5 minutes"
      }
    }
  }
}
```

---

## Безопасность аудитных логов

### Защита от подделки

```python
import hashlib
import hmac

class AuditLogSigner:
    def __init__(self, secret_key):
        self.secret_key = secret_key.encode()

    def sign_entry(self, entry):
        """Подписываем запись для защиты от подделки"""
        data = json.dumps(entry, sort_keys=True).encode()
        signature = hmac.new(self.secret_key, data, hashlib.sha256).hexdigest()

        return {
            **entry,
            '_metadata': {
                'signature': signature,
                'algorithm': 'HMAC-SHA256'
            }
        }

    def verify_entry(self, entry):
        """Проверяем подпись записи"""
        if '_metadata' not in entry:
            return False

        signature = entry['_metadata']['signature']
        data = json.dumps({k: v for k, v in entry.items() if k != '_metadata'},
                         sort_keys=True).encode()
        expected = hmac.new(self.secret_key, data, hashlib.sha256).hexdigest()

        return hmac.compare_digest(signature, expected)
```

### Контроль доступа

```yaml
# RBAC для аудитных логов
- role: auditor
  permissions:
    - audit_logs:read
    - audit_logs:search
    - audit_logs:export
  deny:
    - audit_logs:delete
    - audit_logs:modify

- role: security_admin
  permissions:
    - audit_logs:*
  deny:
    - audit_logs:delete  # Никто не может удалять
```

---

## Соответствие стандартам

### GDPR
- Записи доступа к персональным данным (Article 30)
- Минимум 6 месяцев хранения
- Право на удаление (право быть забытым)

### HIPAA
- Все доступы к PHI (Protected Health Information)
- Минимум 6 лет хранения
- Integrity controls

### PCI DSS
- Все доступы к карточным данным
- Минимум 1 год хранения, 3 года доступности
- Немодифицируемые логи

### SOX
- Все финансовые операции
- Все изменения в системах
- Минимум 7 лет хранения
- Невозможность изменения/удаления

---

## Процесс анализа инцидента

```bash
#!/bin/bash
# analyze-incident.sh

INCIDENT_TIME="2024-01-15T10:30:00"
USER_ID="suspicious_user_123"

echo "=== Анализ инцидента ==="
echo "Время: $INCIDENT_TIME"
echo "Пользователь: $USER_ID"

echo ""
echo "1. Все действия пользователя за 24 часа:"
esearch "actor_id:$USER_ID AND timestamp:[$INCIDENT_TIME-24h TO $INCIDENT_TIME+24h]"

echo ""
echo "2. Все входы:"
esearch "actor_id:$USER_ID AND event_type:user.login"

echo ""
echo "3. Критические операции:"
esearch "actor_id:$USER_ID AND (data.classification:PII OR data.classification:PCI)"

echo ""
echo "4. Изменения прав доступа:"
esearch "actor_id:$USER_ID AND event_type:permission.*"

echo ""
echo "Анализ завершен. Логи сохранены в incident_$USER_ID.json"
```

---

*Audit Logging - практика систематической записи аудитных событий для безопасности и compliance*
