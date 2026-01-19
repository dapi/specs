# SRO-009 Database Security (Безопасность баз данных)

## Определение

Безопасность баз данных охватывает практики и конфигурации, которые защищают системы баз данных от несанкционированного доступа, утечек данных и вредоносных атак. Это включает аутентификацию, авторизацию, шифрование, аудит и управление безопасной конфигурацией.

## Аутентификация

### Аутентификация PostgreSQL (pg_hba.conf)

```bash
# pg_hba.conf - Host-Based Authentication

# TYPE  DATABASE  USER      ADDRESS         METHOD

# Локальные соединения
local   all       postgres                  peer
local   all       all                       scram-sha-256

# IPv4 локальные соединения
host    all       all       127.0.0.1/32    scram-sha-256

# IPv4 внутренняя сеть
host    all       all       10.0.0.0/8      scram-sha-256

# Соединения репликации
host    replication replicator 10.0.0.0/8   scram-sha-256

# Отклонить все остальные соединения
host    all       all       0.0.0.0/0       reject
```

### Шифрование паролей

```sql
-- PostgreSQL: Использовать scram-sha-256 (рекомендуется)
-- postgresql.conf
password_encryption = scram-sha-256

-- Создать пользователя с зашифрованным паролем
CREATE USER app_user WITH PASSWORD 'secure_password';

-- Проверить шифрование пароля
SELECT rolname, rolpassword FROM pg_authid WHERE rolname = 'app_user';

-- MySQL: Использовать caching_sha2_password (по умолчанию в 8.0+)
CREATE USER 'app_user'@'%' IDENTIFIED WITH caching_sha2_password BY 'secure_password';
```

### Аутентификация на основе сертификатов

```bash
# pg_hba.conf для аутентификации по SSL сертификату
hostssl all       all       10.0.0.0/8      cert clientcert=verify-full

# postgresql.conf настройки SSL
ssl = on
ssl_cert_file = '/etc/postgresql/server.crt'
ssl_key_file = '/etc/postgresql/server.key'
ssl_ca_file = '/etc/postgresql/ca.crt'
ssl_crl_file = '/etc/postgresql/root.crl'
```

### Строка подключения с SSL

```python
# Python с psycopg
import psycopg

conn = psycopg.connect(
    host="db.example.com",
    dbname="mydb",
    user="app_user",
    password="secure_password",
    sslmode="verify-full",
    sslrootcert="/path/to/ca.crt",
    sslcert="/path/to/client.crt",
    sslkey="/path/to/client.key"
)
```

## Авторизация

### Контроль доступа на основе ролей (RBAC)

```sql
-- PostgreSQL: Создание ролей с конкретными привилегиями

-- Роль только для чтения
CREATE ROLE readonly;
GRANT CONNECT ON DATABASE mydb TO readonly;
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readonly;

-- Роль для чтения и записи
CREATE ROLE readwrite;
GRANT CONNECT ON DATABASE mydb TO readwrite;
GRANT USAGE ON SCHEMA public TO readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO readwrite;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO readwrite;

-- Роль приложения (только конкретные таблицы)
CREATE ROLE app_role;
GRANT CONNECT ON DATABASE mydb TO app_role;
GRANT USAGE ON SCHEMA public TO app_role;
GRANT SELECT, INSERT, UPDATE ON users, orders, products TO app_role;
GRANT SELECT ON audit_logs TO app_role;

-- Назначить роли пользователям
CREATE USER app_user WITH PASSWORD 'secure_password';
GRANT app_role TO app_user;

CREATE USER report_user WITH PASSWORD 'secure_password';
GRANT readonly TO report_user;
```

### Row-Level Security (RLS)

```sql
-- Включить RLS на таблице
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Политика: Пользователи видят только свои заказы
CREATE POLICY user_orders_policy ON orders
    FOR ALL
    TO app_role
    USING (user_id = current_setting('app.current_user_id')::int);

-- Политика: Администраторы видят все заказы
CREATE POLICY admin_orders_policy ON orders
    FOR ALL
    TO admin_role
    USING (true);

-- Установить контекст пользователя в приложении
SET app.current_user_id = '123';
SELECT * FROM orders;  -- Показывает только заказы пользователя 123

-- Принудительно включить RLS для владельца таблицы
ALTER TABLE orders FORCE ROW LEVEL SECURITY;
```

### Привилегии на уровне колонок

```sql
-- Предоставить доступ к конкретным колонкам
GRANT SELECT (id, name, email) ON users TO readonly;
GRANT UPDATE (name, email) ON users TO app_role;

-- Запретить доступ к чувствительным колонкам
REVOKE ALL ON users FROM PUBLIC;
GRANT SELECT (id, name) ON users TO public_role;
-- Колонки password_hash и ssn недоступны
```

## Шифрование

### Шифрование данных в покое

```sql
-- PostgreSQL: расширение pgcrypto для шифрования колонок
CREATE EXTENSION pgcrypto;

-- Шифровать чувствительные данные
CREATE TABLE users (
    id serial PRIMARY KEY,
    email varchar(255),
    ssn_encrypted bytea  -- Зашифрованный SSN
);

-- Вставка с шифрованием
INSERT INTO users (email, ssn_encrypted)
VALUES (
    'user@example.com',
    pgp_sym_encrypt('123-45-6789', 'encryption_key')
);

-- Выборка с расшифровкой
SELECT
    email,
    pgp_sym_decrypt(ssn_encrypted, 'encryption_key') AS ssn
FROM users;
```

### Transparent Data Encryption (TDE)

```yaml
# PostgreSQL с LUKS шифрованием диска
# /etc/crypttab
pgdata /dev/sdb1 /etc/keys/pgdata.key luks

# Монтировать зашифрованный том
mount /dev/mapper/pgdata /var/lib/postgresql/data

# MySQL Enterprise TDE
# my.cnf
[mysqld]
early-plugin-load=keyring_file.so
keyring_file_data=/var/lib/mysql-keyring/keyring

# Зашифровать tablespace
ALTER TABLESPACE ts1 ENCRYPTION='Y';
```

### Шифрование при передаче

```sql
-- PostgreSQL: Принудительно SSL соединения
-- postgresql.conf
ssl = on
ssl_min_protocol_version = 'TLSv1.3'
ssl_prefer_server_ciphers = on
ssl_ciphers = 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384'

-- pg_hba.conf: Требовать SSL
hostssl all all 0.0.0.0/0 scram-sha-256

-- MySQL: Требовать SSL
CREATE USER 'app_user'@'%' REQUIRE SSL;
-- Или требовать X509 сертификат
CREATE USER 'app_user'@'%' REQUIRE X509;
```

## Предотвращение SQL-инъекций

### Параметризованные запросы

```python
# Python - ПРАВИЛЬНО: Параметризованный запрос
import psycopg

def get_user(conn, user_id):
    cursor = conn.cursor()
    cursor.execute(
        "SELECT * FROM users WHERE id = %s",
        (user_id,)
    )
    return cursor.fetchone()

# Python - НЕПРАВИЛЬНО: Конкатенация строк (уязвимо к SQL-инъекции)
def get_user_unsafe(conn, user_id):
    cursor = conn.cursor()
    # НИКОГДА НЕ ДЕЛАЙТЕ ТАК!
    cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
    return cursor.fetchone()
```

```ruby
# Rails - ПРАВИЛЬНО: Использование ActiveRecord правильно
User.where(email: params[:email])
User.find_by(id: params[:id])

# Rails - НЕПРАВИЛЬНО: Интерполяция строк (уязвимо к SQL-инъекции)
# НИКОГДА НЕ ДЕЛАЙТЕ ТАК!
User.where("email = '#{params[:email]}'")
```

### Валидация входных данных

```python
# Валидировать входные данные перед использованием в запросах
import re

def validate_email(email):
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    if not re.match(pattern, email):
        raise ValueError("Неверный формат email")
    return email

def validate_user_id(user_id):
    try:
        return int(user_id)
    except (ValueError, TypeError):
        raise ValueError("Неверный ID пользователя")
```

## Аудит

### Логирование аудита PostgreSQL

```sql
-- Включить логирование
-- postgresql.conf
log_statement = 'all'  -- или 'ddl', 'mod'
log_duration = on
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_connections = on
log_disconnections = on

-- Расширение pgAudit для детального аудита
CREATE EXTENSION pgaudit;

-- postgresql.conf
pgaudit.log = 'read, write, ddl'
pgaudit.log_catalog = off
pgaudit.log_parameter = on
pgaudit.log_statement_once = on

-- Настройки аудита для конкретной роли
ALTER ROLE app_user SET pgaudit.log = 'write';
```

### Аудит на уровне приложения

```sql
-- Таблица аудита
CREATE TABLE audit_log (
    id bigserial PRIMARY KEY,
    table_name varchar(100) NOT NULL,
    record_id bigint NOT NULL,
    action varchar(10) NOT NULL,  -- INSERT, UPDATE, DELETE
    old_values jsonb,
    new_values jsonb,
    user_id int,
    ip_address inet,
    user_agent text,
    created_at timestamptz DEFAULT now()
);

-- Функция триггера аудита
CREATE OR REPLACE FUNCTION audit_trigger_func()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, record_id, action, new_values, user_id)
        VALUES (TG_TABLE_NAME, NEW.id, 'INSERT', row_to_json(NEW), current_setting('app.user_id', true)::int);
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, record_id, action, old_values, new_values, user_id)
        VALUES (TG_TABLE_NAME, NEW.id, 'UPDATE', row_to_json(OLD), row_to_json(NEW), current_setting('app.user_id', true)::int);
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, record_id, action, old_values, user_id)
        VALUES (TG_TABLE_NAME, OLD.id, 'DELETE', row_to_json(OLD), current_setting('app.user_id', true)::int);
        RETURN OLD;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Применить триггер аудита
CREATE TRIGGER users_audit
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();
```

## Безопасная конфигурация

### Укрепление PostgreSQL

```ini
# postgresql.conf настройки безопасности

# Сеть
listen_addresses = 'localhost,10.0.0.1'  # Только конкретные IP
port = 5432
max_connections = 100

# Аутентификация
password_encryption = scram-sha-256
authentication_timeout = 60s

# SSL
ssl = on
ssl_min_protocol_version = 'TLSv1.3'

# Права доступа
unix_socket_permissions = 0700

# Логирование
log_connections = on
log_disconnections = on
log_statement = 'ddl'
log_min_duration_statement = 1000  # Логировать медленные запросы

# Удалить ненужные расширения
-- DROP EXTENSION IF EXISTS dblink;
-- DROP EXTENSION IF EXISTS postgres_fdw;
```

### Укрепление MySQL

```ini
# my.cnf настройки безопасности

[mysqld]
# Сеть
bind-address = 127.0.0.1
skip-networking = OFF

# Аутентификация
default_authentication_plugin = caching_sha2_password

# SSL
require_secure_transport = ON
ssl_ca = /etc/mysql/ca.pem
ssl_cert = /etc/mysql/server-cert.pem
ssl_key = /etc/mysql/server-key.pem
tls_version = TLSv1.3

# Безопасность
local_infile = OFF
secure_file_priv = /var/lib/mysql-files
skip-symbolic-links = ON

# Логирование
general_log = OFF
log_error = /var/log/mysql/error.log
slow_query_log = ON
```

## Управление секретами

### Переменные окружения

```bash
# Хранить секреты в переменных окружения
export DB_PASSWORD="secure_password"
export DB_SSL_CERT="/path/to/cert.pem"

# Приложение читает из окружения
DATABASE_URL=postgresql://user:${DB_PASSWORD}@host:5432/dbname?sslmode=verify-full
```

### Интеграция с HashiCorp Vault

```python
# Python с hvac (Vault клиент)
import hvac
import psycopg

# Получить учетные данные БД из Vault
client = hvac.Client(url='https://vault.example.com:8200')
client.token = os.environ['VAULT_TOKEN']

# Прочитать учетные данные базы данных
secret = client.secrets.database.generate_credentials('my-role')
username = secret['data']['username']
password = secret['data']['password']

# Подключиться с динамическими учетными данными
conn = psycopg.connect(
    host="db.example.com",
    dbname="mydb",
    user=username,
    password=password
)
```

## Мониторинг безопасности

### Правила алертинга

```yaml
# Алертинг Prometheus для событий безопасности
groups:
  - name: database-security
    rules:
      - alert: FailedLoginAttempts
        expr: rate(pg_stat_activity_failed_logins[5m]) > 10
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Высокая частота неудачных попыток входа"

      - alert: UnauthorizedAccessAttempt
        expr: pg_audit_denied_queries_total > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Обнаружена попытка несанкционированного доступа"

      - alert: SSLConnectionsLow
        expr: pg_stat_ssl_connections / pg_stat_activity_connections < 0.95
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Обнаружены не-SSL соединения"

      - alert: PrivilegedUserActivity
        expr: pg_stat_activity_superuser_connections > 1
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Обнаружено несколько суперпользователей"
```

## Best practices

**Делать:**
- Использовать сильные уникальные пароли и регулярно ротировать
- Реализовать принцип наименьших привилегий
- Включить SSL/TLS для всех соединений
- Использовать параметризованные запросы для предотвращения SQL-инъекций
- Включить логирование аудита для чувствительных операций
- Регулярно проверять и отзывать неиспользуемые привилегии
- Поддерживать ПО БД обновленным с патчами безопасности
- Использовать row-level security для мультитенантных приложений

**Не делать:**
- Хранить пароли в открытом виде
- Использовать суперпользователя для приложений
- Разрешать удаленные подключения без SSL
- Отключать аутентификацию для удобства
- Выдавать избыточные привилегии
- Игнорировать неудачные попытки входа
- Хранить ключи шифрования в базе данных
- Использовать устаревшие версии TLS

## Переменные окружения

```bash
# Аутентификация
DB_AUTH_METHOD=scram-sha-256
DB_SSL_MODE=verify-full
DB_SSL_CERT=/etc/ssl/certs/db-client.crt
DB_SSL_KEY=/etc/ssl/private/db-client.key
DB_SSL_CA=/etc/ssl/certs/ca.crt

# Шифрование
DB_ENCRYPTION_KEY_PATH=/etc/secrets/db-encryption-key
DB_TDE_ENABLED=true

# Аудит
DB_AUDIT_ENABLED=true
DB_AUDIT_LOG_PATH=/var/log/db-audit/
DB_AUDIT_LEVEL=write,ddl

# Мониторинг безопасности
DB_MAX_FAILED_LOGINS=5
DB_LOCKOUT_DURATION_MINUTES=30
```

## Дополнительные ресурсы

- [PostgreSQL Security](https://www.postgresql.org/docs/current/security.html)
- [pgAudit Extension](https://github.com/pgaudit/pgaudit)
- [MySQL Security Guide](https://dev.mysql.com/doc/refman/8.0/en/security.html)
- [OWASP SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [HashiCorp Vault Database Secrets](https://www.vaultproject.io/docs/secrets/databases)
