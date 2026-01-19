# SRS-018 Secrets Management

Secrets Management is the practice of securely storing, transmitting, and managing sensitive data (secrets) such as API keys, passwords, certificates, and access tokens.

## What is considered secrets?

- API keys and access tokens
- Database passwords
- TLS/SSL certificates and private keys
- SSH keys
- Encryption keys
- OAuth client secrets
- Temporary tokens (session tokens)
- Connection strings

---

## Anti-patterns (What NOT to do) ❌

### ❌ Storing in code
```python
# DON'T DO THIS!
api_key = "sk-1234567890abcdef"  # Hardcoded in code
db_password = "super-secret-pass"  # In Git repository
```

**Risks:**
- Forever in Git history
- Accessible to everyone who sees the code
- Requires rebuilding to change

### ❌ Storing in configuration files without encryption
```yaml
# config.yml - DON'T DO THIS!
database:
  password: "my-secret-password"
  host: "db.example.com"
```

### ❌ Sending via insecure channels

- Email
- Slack/Teams
- Messengers
- Including in logs
- URL parameters
- Query parameters
- Command line parameters

### ❌ Shared secrets
- Using one API key for all developers
- Common SSH key for all servers
- "admin/admin" credentials

---

## Best practices ✅

### 1. Using environment variables (basic level)

```bash
# Setting variables on host
export DB_PASSWORD="{{secret_value}}"
```

```python
import os

# Using environment variables in application
db_password = os.environ.get('DB_PASSWORD')

# Creating derived tokens with limited permissions
def create_api_token():
    base_key = os.environ.get('API_MASTER_KEY')
    return generate_limited_token(base_key, permissions=['read'])
```

**⚠️ Important:** Environment variables - basic level, but not the most secure for production.

### 2. Using files mounted at startup

```bash
# Running container with secret files
docker run -d \
  --env DB_PASSWORD_FILE=/run/secrets/db_password \
  --mount type=bind,source=/var/secrets/db_password,target=/run/secrets/db_password,readonly \
  myapp
```

```python
# Reading from file
def read_secret_from_file(path):
    try:
        with open(path, 'r') as f:
            return f.read().strip()
    except FileNotFoundError:
        raise Exception(f"Secret file not found: {path}")

db_password = read_secret_from_file('/run/secrets/db_password')
```

**Pros:**
- File system can be mounted with `readonly` permissions
- Operating system controls access
- Compatible with Docker secrets, Kubernetes secrets

### 3. Secret management systems (recommended)

#### HashiCorp Vault

```bash
# Setting secret
echo -n "super-secret" | vault kv put secret/myapp/db_password value=-

# Reading secret
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
        # Creates temporary database credentials
        response = self.client.secrets.database.generate_credentials(role)
        return response['data']

# Usage
vault = VaultSecretsManager('http://vault:8200', 's.token')
db_pass = vault.get_secret('myapp', 'db_password')
```

**Vault capabilities:**
- Dynamic secrets (generating temporary passwords)
- Automatic rotation
- Secret leasing (lease-based access)
- Audit logging
- Multi-tenant security
- Backup and recovery

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

# Usage
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

# Usage
gsm = GoogleSecretManager('my-project')
api_key = gsm.get_secret('prod-api-key')
```

### 4. Kubernetes Secrets (for K8s environments)

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

**Important:** Use `sealed-secrets` for GitOps:
```bash
# Encrypt secret for storing in Git
kubeseal < secret.yaml > sealed-secret.yaml
```

### 5. Docker Secrets (for Docker Swarm)

```bash
# Creating secret
echo "my-secret-password" | docker secret create db_password -

# Using in service
docker service create \
  --name myapp \
  --secret db_password \
  --env DB_PASSWORD_FILE=/run/secrets/db_password \
  myapp:latest
```

---

## Secret rotation process

### Automatic rotation

```python
class SecretRotator:
    def __init__(self, secrets_manager):
        self.secrets_manager = secrets_manager

    def rotate_database_password(self):
        """Rotate DB password with zero downtime"""
        # 1. Generate new password
        new_password = generate_secure_password()

        # 2. Create new user with same permissions
        db.create_user('app_user_new', new_password)
        db.grant_permissions('app_user_new', 'app_user')

        # 3. Update secret in storage
        self.secrets_manager.update_secret('db_password', new_password)

        # 4. Restart applications with new password
        orchestrator.restart_services('app', rolling=True)

        # 5. Check that everything works
        health_check.wait_all_healthy()

        # 6. Delete old user (after some time)
        schedule.delay(7.days).run(db.drop_user, 'app_user')
```

### Manual rotation during compromise

```bash
#!/bin/bash
# emergency-rotate.sh

echo "🚨 EMERGENCY SECRET ROTATION"

# Stop all services
kubectl scale deployment --all --replicas=0

# Generate new secrets
vault write -f auth/token/roles/app
vault write -f database/rotate-role/db-app

# Update all applications
terraform apply -var="emergency_rotation=true"

# Start services back
kubectl scale deployment --all --replicas=3
```

---

## Configuration via environment variables

```bash
# Example configuration
SECRETS_PROVIDER=vault              # vault, aws, gcp, azure, file
VAULT_ADDR=https://vault.example.com
VAULT_TOKEN=s.token
VAULT_NAMESPACE=production
VAULT_SECRET_PATH=secret/data/myapp

# For development
SECRETS_PROVIDER=file
SECRETS_FILE_PATH=/run/secrets
```

---

## Best practices ✅

### Storage
- ✅ Use specialized secret management systems (Vault, AWS/GCP/Azure Secrets Manager)
- ✅ Store secrets separately from code and configuration
- ✅ Encrypt secrets at rest and in transit
- ✅ Use different keys for different environments (dev, staging, prod)
- ✅ Regularly rotate secrets (automatically or on schedule)
- ✅ Use short-lived temporary secrets (TTL)

### Access
- ✅ Use principle of least privilege (least privilege)
- ✅ Audit log all secret operations
- ✅ Multi-factor authentication for secret access
- ✅ Service-to-service authentication (mTLS)
- ✅ RBAC for access management

### In applications
- ✅ NEVER log secrets
- ✅ Clear secrets from memory after use
- ✅ Pass secrets only through encrypted channels
- ✅ Use environment-specific secrets
- ✅ DO NOT pass secrets via CLI arguments
- ✅ Handle secret access errors

### Monitoring
```python
# Example metrics
metrics.counter('secrets.accessed', tags={'secret_type': 'db_password'})
metrics.counter('secrets.rotation.success')
metrics.counter('secrets.rotation.failed')
metrics.gauge('secrets.expiring_soon', count_near_expiry())
```

---

## How NOT to get secrets ❌

❌ **Don't:**
```python
# DON'T pass via command line
subprocess.run(['app', '--password', password])  # Visible in ps aux

# DON'T use shared secrets
default_api_key = "sk-default-12345"  # Doesn't change between clients

# DON'T commit to repository
# .env file committed to git

# DON'T log
logger.info(f"Connecting to DB with password {db_pass}")

# DON'T send via HTTP without TLS
requests.post('http://insecure.com/api', data={'key': api_key})

# DON'T store in plain text
with open('/tmp/secrets.txt', 'w') as f:
    f.write(f"password={password}")  # Not encrypted!
```

---

## Metrics and monitoring

```python
# Metrics for secrets
SECRETS_RELATED_METRICS = {
    'secrets.access.success': 'Successful secret access',
    'secrets.access.denied': 'Access denied',
    'secrets.rotation.success': 'Successful rotation',
    'secrets.rotation.failed': 'Rotation failed',
    'secrets.expiry.soon': 'Secret expires in less than 7 days',
    'secrets.lease_time': 'Secret lifetime',
}
```

---

## Security audit

```bash
# Checking for secret leaks
git log --all -p -S 'AKIA'  # AWS Access Keys
git log --all -p -S 'sk_live'  # Stripe keys
git log --all -p -S 'BEGIN RSA PRIVATE KEY'  # Private keys

# Tools
# - git-secrets
# - truffleHog
# - git-leaks
```

---

## Incident process: secret leak

```bash
#!/bin/bash
# incident-secret-leak.sh

echo "🚨 INCIDENT: SECRET LEAK DETECTED"

# 1. Revoke secret
vault lease revoke -prefix secret/data/production/app

# 2. Generate new
vault kv put secret/production/app/new_key value=$(generate_secret)

# 3. Update all services
kubectl rollout restart deployment/app

# 4. Analyze the leak
git log --all -p | grep -C 5 "$LEAKED_SECRET"

# 5. Remove from Git history (dangerous!)
# Use BFG Repo-Cleaner or git-filter-repo
```

---

## Solution comparison

| Solution | Complexity | Cost | Best For | Management |
|---------|-----------|-----------|----------|---------------------|
| Environment Variables | Low | Free | Development, simple applications | Self-managed |
| Files Mounted | Medium | Free | Containers, Kubernetes | Self-managed |
| HashiCorp Vault | High | Free / Enterprise | Large scale, on-premise | Self-managed |
| AWS Secrets Manager | Medium | Paid per secret | AWS workloads | AWS-managed |
| GCP Secret Manager | Medium | Paid per secret | GCP workloads | GCP-managed |
| Azure Key Vault | Medium | Paid per operation | Azure workloads | Azure-managed |
| Docker Secrets | Medium | Free | Docker Swarm | Self-managed |
| Kubernetes Secrets | Medium | Free | Kubernetes only | Self-managed |
| Sealed Secrets + GitOps | Medium | Free | Kubernetes + GitOps | Self-managed |

---

*Secrets Management - practice of secure storage and management of sensitive data*
