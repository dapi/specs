# SRO-009 Database Security

## Definition

Database security encompasses practices and configurations that protect database systems from unauthorized access, data breaches, and malicious attacks. This includes authentication, authorization, encryption, auditing, and secure configuration management.

## Authentication

### PostgreSQL Authentication (pg_hba.conf)

```bash
# pg_hba.conf - Host-Based Authentication

# TYPE  DATABASE  USER      ADDRESS         METHOD

# Local connections
local   all       postgres                  peer
local   all       all                       scram-sha-256

# IPv4 local connections
host    all       all       127.0.0.1/32    scram-sha-256

# IPv4 internal network
host    all       all       10.0.0.0/8      scram-sha-256

# Replication connections
host    replication replicator 10.0.0.0/8   scram-sha-256

# Reject all other connections
host    all       all       0.0.0.0/0       reject
```

### Password Encryption

```sql
-- PostgreSQL: Use scram-sha-256 (recommended)
-- postgresql.conf
password_encryption = scram-sha-256

-- Create user with encrypted password
CREATE USER app_user WITH PASSWORD 'secure_password';

-- Verify password encryption
SELECT rolname, rolpassword FROM pg_authid WHERE rolname = 'app_user';

-- MySQL: Use caching_sha2_password (default in 8.0+)
CREATE USER 'app_user'@'%' IDENTIFIED WITH caching_sha2_password BY 'secure_password';
```

### Certificate-Based Authentication

```bash
# pg_hba.conf for SSL certificate auth
hostssl all       all       10.0.0.0/8      cert clientcert=verify-full

# postgresql.conf SSL settings
ssl = on
ssl_cert_file = '/etc/postgresql/server.crt'
ssl_key_file = '/etc/postgresql/server.key'
ssl_ca_file = '/etc/postgresql/ca.crt'
ssl_crl_file = '/etc/postgresql/root.crl'
```

### Connection String with SSL

```python
# Python with psycopg
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

## Authorization

### Role-Based Access Control (RBAC)

```sql
-- PostgreSQL: Create roles with specific privileges

-- Read-only role
CREATE ROLE readonly;
GRANT CONNECT ON DATABASE mydb TO readonly;
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readonly;

-- Read-write role
CREATE ROLE readwrite;
GRANT CONNECT ON DATABASE mydb TO readwrite;
GRANT USAGE ON SCHEMA public TO readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO readwrite;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO readwrite;

-- Application role (specific tables only)
CREATE ROLE app_role;
GRANT CONNECT ON DATABASE mydb TO app_role;
GRANT USAGE ON SCHEMA public TO app_role;
GRANT SELECT, INSERT, UPDATE ON users, orders, products TO app_role;
GRANT SELECT ON audit_logs TO app_role;

-- Assign roles to users
CREATE USER app_user WITH PASSWORD 'secure_password';
GRANT app_role TO app_user;

CREATE USER report_user WITH PASSWORD 'secure_password';
GRANT readonly TO report_user;
```

### Row-Level Security (RLS)

```sql
-- Enable RLS on table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see their own orders
CREATE POLICY user_orders_policy ON orders
    FOR ALL
    TO app_role
    USING (user_id = current_setting('app.current_user_id')::int);

-- Policy: Admins can see all orders
CREATE POLICY admin_orders_policy ON orders
    FOR ALL
    TO admin_role
    USING (true);

-- Set user context in application
SET app.current_user_id = '123';
SELECT * FROM orders;  -- Only shows user 123's orders

-- Force RLS for table owner
ALTER TABLE orders FORCE ROW LEVEL SECURITY;
```

### Column-Level Privileges

```sql
-- Grant specific column access
GRANT SELECT (id, name, email) ON users TO readonly;
GRANT UPDATE (name, email) ON users TO app_role;

-- Deny access to sensitive columns
REVOKE ALL ON users FROM PUBLIC;
GRANT SELECT (id, name) ON users TO public_role;
-- password_hash and ssn columns not accessible
```

## Encryption

### Encryption at Rest

```sql
-- PostgreSQL: pgcrypto extension for column encryption
CREATE EXTENSION pgcrypto;

-- Encrypt sensitive data
CREATE TABLE users (
    id serial PRIMARY KEY,
    email varchar(255),
    ssn_encrypted bytea  -- Encrypted SSN
);

-- Insert with encryption
INSERT INTO users (email, ssn_encrypted)
VALUES (
    'user@example.com',
    pgp_sym_encrypt('123-45-6789', 'encryption_key')
);

-- Select with decryption
SELECT
    email,
    pgp_sym_decrypt(ssn_encrypted, 'encryption_key') AS ssn
FROM users;
```

### Transparent Data Encryption (TDE)

```yaml
# PostgreSQL with LUKS disk encryption
# /etc/crypttab
pgdata /dev/sdb1 /etc/keys/pgdata.key luks

# Mount encrypted volume
mount /dev/mapper/pgdata /var/lib/postgresql/data

# MySQL Enterprise TDE
# my.cnf
[mysqld]
early-plugin-load=keyring_file.so
keyring_file_data=/var/lib/mysql-keyring/keyring

# Encrypt tablespace
ALTER TABLESPACE ts1 ENCRYPTION='Y';
```

### Encryption in Transit

```sql
-- PostgreSQL: Force SSL connections
-- postgresql.conf
ssl = on
ssl_min_protocol_version = 'TLSv1.3'
ssl_prefer_server_ciphers = on
ssl_ciphers = 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384'

-- pg_hba.conf: Require SSL
hostssl all all 0.0.0.0/0 scram-sha-256

-- MySQL: Require SSL
CREATE USER 'app_user'@'%' REQUIRE SSL;
-- Or require X509 certificate
CREATE USER 'app_user'@'%' REQUIRE X509;
```

## SQL Injection Prevention

### Parameterized Queries

```python
# Python - CORRECT: Parameterized query
import psycopg

def get_user(conn, user_id):
    cursor = conn.cursor()
    cursor.execute(
        "SELECT * FROM users WHERE id = %s",
        (user_id,)
    )
    return cursor.fetchone()

# Python - WRONG: String concatenation (SQL injection vulnerable)
def get_user_unsafe(conn, user_id):
    cursor = conn.cursor()
    # NEVER DO THIS!
    cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
    return cursor.fetchone()
```

```ruby
# Rails - CORRECT: Using ActiveRecord properly
User.where(email: params[:email])
User.find_by(id: params[:id])

# Rails - WRONG: String interpolation (SQL injection vulnerable)
# NEVER DO THIS!
User.where("email = '#{params[:email]}'")
```

### Input Validation

```python
# Validate input before using in queries
import re

def validate_email(email):
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    if not re.match(pattern, email):
        raise ValueError("Invalid email format")
    return email

def validate_user_id(user_id):
    try:
        return int(user_id)
    except (ValueError, TypeError):
        raise ValueError("Invalid user ID")
```

## Auditing

### PostgreSQL Audit Logging

```sql
-- Enable logging
-- postgresql.conf
log_statement = 'all'  -- or 'ddl', 'mod'
log_duration = on
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_connections = on
log_disconnections = on

-- pgAudit extension for detailed auditing
CREATE EXTENSION pgaudit;

-- postgresql.conf
pgaudit.log = 'read, write, ddl'
pgaudit.log_catalog = off
pgaudit.log_parameter = on
pgaudit.log_statement_once = on

-- Per-role audit settings
ALTER ROLE app_user SET pgaudit.log = 'write';
```

### Application-Level Audit Trail

```sql
-- Audit table
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

-- Audit trigger function
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

-- Apply audit trigger
CREATE TRIGGER users_audit
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();
```

## Secure Configuration

### PostgreSQL Hardening

```ini
# postgresql.conf security settings

# Network
listen_addresses = 'localhost,10.0.0.1'  # Specific IPs only
port = 5432
max_connections = 100

# Authentication
password_encryption = scram-sha-256
authentication_timeout = 60s

# SSL
ssl = on
ssl_min_protocol_version = 'TLSv1.3'

# Permissions
unix_socket_permissions = 0700

# Logging
log_connections = on
log_disconnections = on
log_statement = 'ddl'
log_min_duration_statement = 1000  # Log slow queries

# Remove unnecessary extensions
-- DROP EXTENSION IF EXISTS dblink;
-- DROP EXTENSION IF EXISTS postgres_fdw;
```

### MySQL Hardening

```ini
# my.cnf security settings

[mysqld]
# Network
bind-address = 127.0.0.1
skip-networking = OFF

# Authentication
default_authentication_plugin = caching_sha2_password

# SSL
require_secure_transport = ON
ssl_ca = /etc/mysql/ca.pem
ssl_cert = /etc/mysql/server-cert.pem
ssl_key = /etc/mysql/server-key.pem
tls_version = TLSv1.3

# Security
local_infile = OFF
secure_file_priv = /var/lib/mysql-files
skip-symbolic-links = ON

# Logging
general_log = OFF
log_error = /var/log/mysql/error.log
slow_query_log = ON
```

## Secret Management

### Environment Variables

```bash
# Store secrets in environment variables
export DB_PASSWORD="secure_password"
export DB_SSL_CERT="/path/to/cert.pem"

# Application reads from environment
DATABASE_URL=postgresql://user:${DB_PASSWORD}@host:5432/dbname?sslmode=verify-full
```

### HashiCorp Vault Integration

```python
# Python with hvac (Vault client)
import hvac
import psycopg

# Get database credentials from Vault
client = hvac.Client(url='https://vault.example.com:8200')
client.token = os.environ['VAULT_TOKEN']

# Read database credentials
secret = client.secrets.database.generate_credentials('my-role')
username = secret['data']['username']
password = secret['data']['password']

# Connect with dynamic credentials
conn = psycopg.connect(
    host="db.example.com",
    dbname="mydb",
    user=username,
    password=password
)
```

## Monitoring Security

### Alerting Rules

```yaml
# Prometheus alerting for security events
groups:
  - name: database-security
    rules:
      - alert: FailedLoginAttempts
        expr: rate(pg_stat_activity_failed_logins[5m]) > 10
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "High rate of failed login attempts"

      - alert: UnauthorizedAccessAttempt
        expr: pg_audit_denied_queries_total > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Unauthorized database access attempt detected"

      - alert: SSLConnectionsLow
        expr: pg_stat_ssl_connections / pg_stat_activity_connections < 0.95
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Non-SSL connections detected"

      - alert: PrivilegedUserActivity
        expr: pg_stat_activity_superuser_connections > 1
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Multiple superuser connections detected"
```

## Best Practices

**Do:**
- Use strong, unique passwords and rotate regularly
- Implement principle of least privilege
- Enable SSL/TLS for all connections
- Use parameterized queries to prevent SQL injection
- Enable audit logging for sensitive operations
- Regularly review and revoke unused privileges
- Keep database software updated with security patches
- Use row-level security for multi-tenant applications

**Don't:**
- Store passwords in plain text
- Use superuser accounts for applications
- Allow remote connections without SSL
- Disable authentication for convenience
- Grant excessive privileges
- Ignore failed login attempts
- Store encryption keys in the database
- Use outdated TLS versions

## Environment Variables

```bash
# Authentication
DB_AUTH_METHOD=scram-sha-256
DB_SSL_MODE=verify-full
DB_SSL_CERT=/etc/ssl/certs/db-client.crt
DB_SSL_KEY=/etc/ssl/private/db-client.key
DB_SSL_CA=/etc/ssl/certs/ca.crt

# Encryption
DB_ENCRYPTION_KEY_PATH=/etc/secrets/db-encryption-key
DB_TDE_ENABLED=true

# Auditing
DB_AUDIT_ENABLED=true
DB_AUDIT_LOG_PATH=/var/log/db-audit/
DB_AUDIT_LEVEL=write,ddl

# Security monitoring
DB_MAX_FAILED_LOGINS=5
DB_LOCKOUT_DURATION_MINUTES=30
```

## Additional Resources

- [PostgreSQL Security](https://www.postgresql.org/docs/current/security.html)
- [pgAudit Extension](https://github.com/pgaudit/pgaudit)
- [MySQL Security Guide](https://dev.mysql.com/doc/refman/8.0/en/security.html)
- [OWASP SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [HashiCorp Vault Database Secrets](https://www.vaultproject.io/docs/secrets/databases)
