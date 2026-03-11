# SRS-023 API Keys Authentication

## Definition

API Keys are shared secrets passed in each request (typically via an HTTP header) to identify and authenticate the calling service. The simplest authentication mechanism — no PKI or auth server required.

## When to Use

| Scenario | Recommendation |
|----------|----------------|
| Simple third-party integrations | ✅ Acceptable |
| Legacy system compatibility | ✅ Acceptable |
| Developer/testing environments | ✅ Acceptable |
| Internal service-to-service | ❌ Use mTLS or JWT |
| High security requirements | ❌ Use mTLS |
| Fine-grained access control | ❌ Use OAuth 2.0 |

## Key Format

Keys must be:
* Cryptographically random — at least 256 bits of entropy
* Unpredictable — not derived from service name or timestamp
* Prefixed for easy identification in logs: `prod_`, `test_`, `sk_live_`

```python
import secrets

def generate_api_key(prefix: str = "sk") -> str:
    # 32 bytes = 256 bits of entropy, base64url encoded
    random_part = secrets.token_urlsafe(32)
    return f"{prefix}_{random_part}"

# Example output: sk_Xt4mP9qR2v...
```

## Transmission

Keys MUST be sent in an HTTP header, never in the URL:

```bash
# Correct
curl -H "X-API-Key: sk_Xt4mP9qR2v..." https://api.example.com/data

# Wrong — key appears in server logs and browser history
curl "https://api.example.com/data?api_key=sk_Xt4mP9qR2v..."
```

## Storage

Keys MUST be stored hashed, never in plaintext:

```python
import hashlib
import hmac
import os
from pathlib import Path

def load_api_key_pepper() -> str:
    path = os.getenv("SERVICE_AUTH_API_KEY_PEPPER_PATH", "/etc/secrets/api_key_pepper")
    return Path(path).read_text().strip()

def hash_api_key(raw_key: str) -> str:
    """Hash key for storage. Use SHA-256 with a pepper."""
    pepper = load_api_key_pepper()
    return hashlib.sha256(f"{pepper}{raw_key}".encode()).hexdigest()

def verify_api_key(raw_key: str, stored_hash: str) -> bool:
    """Timing-safe comparison to prevent timing attacks."""
    expected = hash_api_key(raw_key)
    return hmac.compare_digest(expected, stored_hash)
```

## Implementation

### Flask Decorator

```python
from functools import wraps
from flask import request, jsonify, g

def require_api_key(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        raw_key = request.headers.get("X-API-Key")

        if not raw_key:
            return jsonify({"error": "API key required"}), 401

        service = lookup_service_by_key(raw_key)
        if service is None:
            return jsonify({"error": "Invalid API key"}), 403

        g.service_name = service.name
        g.service_id = service.id
        return f(*args, **kwargs)

    return decorated_function

def lookup_service_by_key(raw_key: str):
    """Look up service by hashed key. Returns None if not found."""
    key_hash = hash_api_key(raw_key)
    return Service.query.filter_by(
        api_key_hash=key_hash,
        is_active=True
    ).first()

@app.route("/api/data")
@require_api_key
def get_data():
    return jsonify({"service": g.service_name, "data": "..."})
```

### FastAPI Middleware

```python
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()

@app.middleware("http")
async def api_key_middleware(request: Request, call_next):
    if request.url.path in ["/health", "/live", "/ready"]:
        return await call_next(request)

    raw_key = request.headers.get("X-API-Key")
    if not raw_key:
        raise HTTPException(status_code=401, detail="API key required")

    service = await lookup_service_by_key_async(raw_key)
    if service is None:
        raise HTTPException(status_code=403, detail="Invalid API key")

    request.state.service_name = service.name
    return await call_next(request)
```

## Key Rotation

API keys MUST support rotation without service downtime:

```python
# Database schema supports multiple active keys per service
# to allow rolling rotation
class ApiKey(Base):
    __tablename__ = "api_keys"
    id = Column(UUID, primary_key=True)
    service_id = Column(UUID, ForeignKey("services.id"), nullable=False)
    key_hash = Column(String, nullable=False, unique=True)
    key_prefix = Column(String(8))   # e.g., "sk_Xt4m" for identification
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime)
    expires_at = Column(DateTime)    # Optional hard expiry
    last_used_at = Column(DateTime)  # For auditing old keys
```

### Rotation Procedure

```
1. Generate new key → issue to service
2. Service deploys new key (both old and new are active)
3. Verify new key is working in production
4. Deactivate old key (set is_active=False)
5. Delete old key after 7-day grace period
```

## Configuration

```
SERVICE_AUTH_API_KEY_HEADER=X-API-Key
SERVICE_AUTH_API_KEY_PREFIX=sk
SERVICE_AUTH_API_KEY_ROTATION_ENABLED=true
SERVICE_AUTH_API_KEY_MAX_ACTIVE_KEYS=2          # Per service during rotation
SERVICE_AUTH_API_KEY_PEPPER_PATH=/etc/secrets/api_key_pepper
```

## Monitoring

```
service_api_key_requests_total{result="success|failure"} (counter)
service_api_key_invalid_attempts_total{service="name"} (counter)
service_api_key_active_keys_total{service="name"} (gauge)
service_api_key_last_rotation_timestamp{service="name"} (gauge)
```

Alert on: more than 5 invalid attempts within 1 minute from the same IP (possible brute force).

## Best Practices

✅ **Do**
* Generate keys with at least 256 bits of entropy
* Store only the hash, never the raw key
* Use timing-safe comparison (`hmac.compare_digest`)
* Support key rotation with grace period
* Log key usage with `key_prefix` (not the full key)
* Set expiry dates on keys used for external integrations
* Transmit only over HTTPS

❌ **Don't**
* Use API keys for internal service-to-service authentication
* Store raw keys in any database or log
* Accept keys in URL query parameters
* Share one key across multiple services
* Allow keys to live indefinitely without rotation

## Pros and Cons

**Pros:**
* Simple to implement and use
* No infrastructure dependencies (no PKI, no auth server)
* Low overhead per request
* Easy to issue to external partners

**Cons:**
* Key does not auto-expire (must be explicitly revoked)
* No built-in scope or permission system
* Transmitted in every request (higher exposure surface)
* Usage tracking requires custom implementation

## Additional Resources

* [OWASP API Security — Broken Authentication](https://owasp.org/www-project-api-security/)
* [RFC 7235 — HTTP Authentication](https://tools.ietf.org/html/rfc7235)
