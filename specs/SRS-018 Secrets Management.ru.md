# SRS-018 Secrets Management (Управление секретами)

Secrets Management - это практика безопасного хранения, передачи и управления чувствительными данными (секретами), такими как API ключи, пароли, сертификаты и токены доступа.

## Что считается секретами?

- API ключи и токены доступа
- Пароли к базам данных
- TLS/SSL сертификаты и приватные ключи
- SSH ключи
- Ключи шифрования
- OAuth клиентские секреты
- Временные токены (session tokens)
- Connection strings

---

## Антипаттерны (Что НЕ делать) ❌

### ❌ Хранение в коде
```python
# НЕ ДЕЛАТЬ!
api_key = "sk-1234567890abcdef"  # Жестко прописано в коде
db_password = "super-secret-pass"  # В репозитории Git
```

**Риски:**
- Попадание в Git history навсегда
- Доступно всем, кто видит код
- Требует пересборки для изменения

### ❌ Хранение в конфигурационных файлах без шифрования
```yaml
# config.yml - НЕ ДЕЛАТЬ!
database:
  password: "my-secret-password"
  host: "db.example.com"
```

### ❌ Отправка через небезопасные каналы

- Email
- Slack/Teams
- Messengers
- Включение в логи
- URL parameters
- Query parameters
- Параметры командной строки

### ❌ Общие секреты
- Использование одного API ключа всеми разработчиками
- Общий SSH ключ на все серверы
- "admin/admin" учетные данные

---

## Хорошие практики ✅

### 1. Использование переменных окружения (базовый уровень)

```bash
# Устанавливаем переменные на хосте
export DB_PASSWORD="{{secret_value}}"
```

```python
import os

# В приложении используем переменные окружения
db_password = os.environ.get('DB_PASSWORD')

# Создаем производные токены с ограниченными правами
def create_api_token():
    base_key = os.environ.get('API_MASTER_KEY')
    return generate_limited_token(base_key, permissions=['read'])
```

**⚠️ Важно:** Переменные окружения - базовый уровень, но не самый безопасный для production.

### 2. Использование файлов, монтируемых при запуске

```bash
# Запуск контейнера с файлами секретов
docker run -d \
  --env DB_PASSWORD_FILE=/run/secrets/db_password \
  --mount type=bind,source=/var/secrets/db_password,target=/run/secrets/db_password,readonly \
  myapp
```

```python
# Чтение из файла
def read_secret_from_file(path):
    try:
        with open(path, 'r') as f:
            return f.read().strip()
    except FileNotFoundError:
        raise Exception(f"Secret file not found: {path}")

db_password = read_secret_from_file('/run/secrets/db_password')
```

**Плюсы:**
- Файловая система может быть монтирована с правами `readonly`
- Операционная система контролирует доступ
- Совместимость с Docker secrets, Kubernetes secrets

### 3. Системы управления секретами (рекомендуется)

#### HashiCorp Vault

```bash
# Установка секрета
echo -n "super-secret" | vault kv put secret/myapp/db_password value=-

# Чтение секрета
vault kv get -field=value secret/myapp/db_password
```

```python
import hvac

class VaultSecretsManager:
    def __init__(self, vault_url, token):
        self.client = hvac.Client(url=vault_url, token=token)

    def get_secret(self, path, key):
        response = self.client.secrets.kv.v2.read_secret_version(path=path)
        return response['data']['data'][key]

    def create_dynamic_secret(self, role):
        # Создает временные учетные данные БД
        response = self.client.secrets.database.generate_credentials(role)
        return response['data']

# Использование
vault = VaultSecretsManager('http://vault:8200', 's.token')
db_pass = vault.get_secret('myapp', 'db_password')
```

**Возможности Vault:**
- Динамические секреты (генерация временных паролей)
- Автоматический ротация
- Утечка секретов (lease-based access)
- Audit logging
- Мульти-tenant безопасность
- Резервное копирование и восстановление

#### AWS Secrets Manager

```python
import boto3
from botocore.exceptions import ClientError

class AWSSecretsManager:
    def __init__(self):
        self.client = boto3.client('secretsmanager')

    def get_secret(self, secret_name):
        try:
            response = self.client.get_secret_value(SecretId=secret_name)
            return response['SecretString']
        except ClientError as e:
            raise Exception(f"Failed to get secret: {e}")

# Использование
aws_sm = AWSSecretsManager()
db_password = aws_sm.get_secret('prod/db/password')
```

#### Google Secret Manager

```python
from google.cloud import secretmanager

class GoogleSecretManager:
    def __init__(self, project_id):
        self.client = secretmanager.SecretManagerServiceClient()
        self.project_id = project_id

    def get_secret(self, secret_id, version='latest'):
        name = f"projects/{self.project_id}/secrets/{secret_id}/versions/{version}"
        response = self.client.access_secret_version(name=name)
        return response.payload.data.decode('UTF-8')

# Использование
gsm = GoogleSecretManager('my-project')
api_key = gsm.get_secret('prod-api-key')
```

### 4. Kubernetes Secrets (для K8s окружений)

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  password: c3VwZXItc2VjcmV0  # base64 encoded
```

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        volumeMounts:
        - name: secrets
          mountPath: /run/secrets
          readOnly: true
      volumes:
      - name: secrets
        secret:
          secretName: db-credentials
```

**Важно:** Используйте `sealed-secrets` для GitOps:
```bash
# Шифруем секрет для хранения в Git
kubeseal < secret.yaml > sealed-secret.yaml
```

### 5. Docker Secrets (для Docker Swarm)

```bash
# Создание секрета
echo "my-secret-password" | docker secret create db_password -

# Использование в сервисе
docker service create \
  --name myapp \
  --secret db_password \
  --env DB_PASSWORD_FILE=/run/secrets/db_password \
  myapp:latest
```

---

## Процесс ротации секретов

### Автоматическая ротация

```python
class SecretRotator:
    def __init__(self, secrets_manager):
        self.secrets_manager = secrets_manager

    def rotate_database_password(self):
        """Ротация пароля БД с нулевым downtime"""
        # 1. Генерируем новый пароль
        new_password = generate_secure_password()

        # 2. Создаем нового пользователя с теми же правами
        db.create_user('app_user_new', new_password)
        db.grant_permissions('app_user_new', 'app_user')

        # 3. Обновляем секрет в хранилище
        self.secrets_manager.update_secret('db_password', new_password)

        # 4. Перезапускаем приложения с новым паролем
        orchestrator.restart_services('app', rolling=True)

        # 5. Проверяем, что все работает
        health_check.wait_all_healthy()

        # 6. Удаляем старого пользователя (через некоторое время)
        schedule.delay(7.days).run(db.drop_user, 'app_user')
```

### Ручная ротация при компрометации

```bash
#!/bin/bash
# emergency-rotate.sh

echo "🚨 EMERGENCY SECRET ROTATION"

# Останавливаем все сервисы
kubectl scale deployment --all --replicas=0

# Генерируем новые секреты
vault write -f auth/token/roles/app
vault write -f database/rotate-role/db-app

# Обновляем все приложения
terraform apply -var="emergency_rotation=true"

# Запускаем сервисы обратно
kubectl scale deployment --all --replicas=3
```

---

## Конфигурация через переменные окружения

```bash
# Пример конфигурации
SECRETS_PROVIDER=vault              # vault, aws, gcp, azure, file
VAULT_ADDR=https://vault.example.com
VAULT_TOKEN=s.token
VAULT_NAMESPACE=production
VAULT_SECRET_PATH=secret/data/myapp

# Для разработки
SECRETS_PROVIDER=file
SECRETS_FILE_PATH=/run/secrets
```

---

## Best practices ✅

### Хранение
- ✅ Использовать специализированные системы управления секретами (Vault, AWS/GCP/Azure Secrets Manager)
- ✅ Хранить секреты отдельно от кода и конфигурации
- ✅ Шифровать секреты в rest и in transit
- ✅ Использовать different keys for different environments (dev, staging, prod)
- ✅ Регулярно ротировать секреты (автоматически или по расписанию)
- ✅ Использовать краткосрочные временные секреты (TTL)

### Доступ
- ✅ Использовать принцип наименьших привилегий (least privilege)
- ✅ Audit log всех операций с секретами
- ✅ Multi-factor authentication для доступа к секретам
- ✅ Service-to-service authentication (mTLS)
- ✅ RBAC для управления доступом

### В приложениях
- ✅ НИКОГДА не логировать секреты
- ✅ Очищать секреты из памяти после использования
- ✅ Передавать секреты только по зашифрованным каналам
- ✅ Использовать environment-specific secrets
- ✅ НЕ передавать секреты через CLI arguments
- ✅ Обрабатывать ошибки доступа к секретам

### Мониторинг
```python
# Пример метрик
metrics.counter('secrets.accessed', tags={'secret_type': 'db_password'})
metrics.counter('secrets.rotation.success')
metrics.counter('secrets.rotation.failed')
metrics.gauge('secrets.expiring_soon', count_near_expiry())
```

---

## Как НЕ получать секреты ❌

❌ **НЕ делать:**
```python
# НЕ передавать в командной строке
subprocess.run(['app', '--password', password])  # Видно в ps aux

# НЕ использовать общие секреты
default_api_key = "sk-default-12345"  # Не меняется между клиентами

# НЕ коммитить в репозиторий
# .env file committed to git

# НЕ логировать
logger.info(f"Connecting to DB with password {db_pass}")

# НЕ передавать по HTTP без TLS
requests.post('http://insecure.com/api', data={'key': api_key})

# НЕ хранить в plain text
with open('/tmp/secrets.txt', 'w') as f:
    f.write(f"password={password}")  # Незашифровано!
```

---

## Метрики и мониторинг

```python
# Метрики для секретов
SECRETS_RELATED_METRICS = {
    'secrets.access.success': 'Успешный доступ к секрету',
    'secrets.access.denied': 'Отказ в доступе',
    'secrets.rotation.success': 'Успешная ротация',
    'secrets.rotation.failed': 'Ошибка ротации',
    'secrets.expiry.soon': 'Секрет истекает менее чем через 7 дней',
    'secrets.lease_time': 'Время жизни секрета',
}
```

---

## Проверка безопасности

```bash
# Проверка на утечку секретов
git log --all -p -S 'AKIA'  # AWS Access Keys
git log --all -p -S 'sk_live'  # Stripe keys
git log --all -p -S 'BEGIN RSA PRIVATE KEY'  # Private keys

# Инструменты
# - git-secrets
# - truffleHog
# - git-leaks
```

---

## Процесс инцидента: утечка секрета

```bash
#!/bin/bash
# incident-secret-leak.sh

echo "🚨 INCIDENT: SECRET LEAK DETECTED"

# 1. Отзываем секрет
vault lease revoke -prefix secret/data/production/app

# 2. Генерируем новый
vault kv put secret/production/app/new_key value=$(generate_secret)

# 3. Обновляем все сервисы
kubectl rollout restart deployment/app

# 4. Анализируем утечку
git log --all -p | grep -C 5 "$LEAKED_SECRET"

# 5. Удаляем из Git history (опасно!)
# Использовать BFG Repo-Cleaner или git-filter-repo
```

---

## Сравнение решений

| Решение | Сложность | Стоимость | Best For | Управление кластером |
|---------|-----------|-----------|----------|---------------------|
| Environment Variables | Низкая | Бесплатно | Разработка, простые приложения | Self-managed |
| Files Mounted | Средняя | Бесплатно | Containers, Kubernetes | Self-managed |
| HashiCorp Vault | Высокая | Бесплатно / Enterprise | Large scale, on-premise | Self-managed |
| AWS Secrets Manager | Средняя | Paid per secret | AWS workloads | AWS-managed |
| GCP Secret Manager | Средняя | Paid per secret | GCP workloads | GCP-managed |
| Azure Key Vault | Средняя | Paid per operation | Azure workloads | Azure-managed |
| Docker Secrets | Средняя | Free | Docker Swarm | Self-managed |
| Kubernetes Secrets | Средняя | Free | Kubernetes only | Self-managed |
| Sealed Secrets + GitOps | Средняя | Free | Kubernetes + GitOps | Self-managed |

---

*Secrets Management - практика безопасного хранения и управления чувствительными данными*
