# SRS-019 Service Authentication

## Definition

Service Authentication is the process of verifying the authenticity of a service making a request to prevent unauthorized access to resources.

## Types of authentication

### 1. mTLS (Mutual TLS)

The most secure way to authenticate between services.

```yaml
# Example on Nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    # Server certificate
    ssl_certificate /etc/ssl/certs/server.crt;
    ssl_certificate_key /etc/ssl/private/server.key;

    # CA to verify client certificates
    ssl_client_certificate /etc/ssl/certs/ca.crt;
    ssl_verify_client on;  # Require client certificate
    ssl_verify_depth 2;

    location / {
        # Pass client info in headers
        proxy_set_header X-SSL-Client-Serial $ssl_client_serial;
        proxy_set_header X-SSL-Client-DN $ssl_client_s_dn;
        proxy_set_header X-SSL-Client-Verify $ssl_client_verify;

        proxy_pass http://app;
    }
}
```

Pros:
* High security
* Service name in certificate
* Impossible to forge

Cons:
* Complexity of certificate management
* TLS handshake overhead
* Need PKI

### 2. JWT Service Tokens

```python
import jwt
import time

# Creating service token
def create_service_token(service_name, private_key):
    payload = {
        "iss": service_name,                    # Issuer
        "sub": service_name,                    # Subject
        "aud": "target-service",                # Audience
        "exp": time.time() + 3600,              # Expiration (1 hour)
        "iat": time.time(),                     # Issued at
        "jti": generate_unique_id(),            # JWT ID (for revocation)
        "scope": ["read", "write"]              # Permissions
    }

    return jwt.encode(payload, private_key, algorithm="RS256")

# Verify service token
def verify_service_token(token, public_key, expected_issuer):
    try:
        payload = jwt.decode(
            token,
            public_key,
            algorithms=["RS256"],
            audience="target-service",
            issuer=expected_issuer
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise Exception("Token expired")
    except jwt.InvalidIssuerError:
        raise Exception("Invalid issuer")
```

Pros:
* Easy to use (HTTP header)
* Can be verified without auth service request
* Built-in expiration time
* Supported by most frameworks

Cons:
* Cannot immediately revoke (without blacklist)
* Need time synchronization
* Token size (1-2KB)

### 3. OAuth 2.0 Client Credentials

```bash
# Request token
curl -X POST https://auth.example.com/oauth/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=service-a" \
  -d "client_secret=secret-key"

# Response
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "read write"
}

# Using token
curl -X GET https://api.example.com/data \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
```

Pros:
* Standard protocol (RFC 6749)
* Ability to revoke tokens
* Centralized management
* Scope support

Cons:
* Additional request for token
* Infrastructure complexity (need auth server)
* Overhead

### 4. API Keys

```python
# Verify API Key
from functools import wraps
import hashlib

def require_api_key(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        api_key = request.headers.get('X-API-Key')

        if not api_key:
            return jsonify({"error": "API key required"}), 401

        # Compare hashes (timing-safe)
        expected_key_hash = os.getenv('SERVICE_A_API_KEY_HASH')
        provided_key_hash = hashlib.sha256(api_key.encode()).hexdigest()

        if not hmac.compare_digest(expected_key_hash, provided_key_hash):
            return jsonify({"error": "Invalid API key"}), 403

        # Key is valid
        g.service_name = "service-a"
        return f(*args, **kwargs)

    return decorated_function

# Usage
@app.route('/api/data')
@require_api_key
def get_data():
    return jsonify({"data": "secret"})
```

Pros:
* Simple
* Low overhead
* Suitable for simple scenarios

Cons:
* API Key doesn't auto-expire
* Need additional revocation list
* Less secure (sent in every request)
* Harder to track usage

## Selection recommendations

| Scenario | Recommendation |
|----------|-------------|
| Service-to-service within cluster | mTLS |
| External services | JWT or OAuth 2.0 |
| Simple integrations | API Keys |
| High security requirements | mTLS + JWT |
| Legacy systems | API Keys |

## Configuration

```
# Service Authentication
SERVICE_AUTH_ENABLED=true
SERVICE_AUTH_TYPE=jwt  # jwt, mtls, oauth2, apikey

# JWT settings
SERVICE_AUTH_JWT_ALGORITHM=RS256
SERVICE_AUTH_JWT_ISSUER=auth-service
SERVICE_AUTH_JWT_AUDIENCE=target-service
SERVICE_AUTH_JWT_EXPIRATION=3600
SERVICE_AUTH_JWT_CLOCK_SKEW=60

# mTLS settings
SERVICE_AUTH_MTLS_VERIFY=true
SERVICE_AUTH_MTLS_CA_CERT_PATH=/etc/ssl/certs/ca.crt
SERVICE_AUTH_MTLS_CERT_PATH=/etc/ssl/certs/client.crt
SERVICE_AUTH_MTLS_KEY_PATH=/etc/ssl/private/client.key
SERVICE_AUTH_MTLS_VERIFY_DEPTH=2

# OAuth2 settings
SERVICE_AUTH_OAUTH2_TOKEN_URL=https://auth.example.com/oauth/token
SERVICE_AUTH_OAUTH2_CLIENT_ID=service-name
SERVICE_AUTH_OAUTH2_CLIENT_SECRET=secret
SERVICE_AUTH_OAUTH2_SCOPES=read,write
SERVICE_AUTH_OAUTH2_CACHE_TOKENS=true
SERVICE_AUTH_OAUTH2_TOKEN_CACHE_TTL=3500

# API Key settings
SERVICE_AUTH_API_KEY_HEADER=X-API-Key
SERVICE_AUTH_API_KEY_PREFIX=  # Optional prefix (e.g., prod_)
SERVICE_AUTH_API_KEY_ROTATION_ENABLED=true
SERVICE_AUTH_API_KEY_OLD_KEYS_CACHE=true
```

## Example implementation in Python (FastAPI)

```python
from fastapi import FastAPI, Depends, HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt

app = FastAPI()
security = HTTPBearer()

# Public key for JWT verification
PUBLIC_KEY = """-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCg...
-----END PUBLIC KEY-----"""

ALLOWED_SERVICES = ["service-a", "service-b", "service-c"]

def verify_service_token(credentials: HTTPAuthorizationCredentials = Security(security)):
    """Verify JWT service token"""
    try:
        token = credentials.credentials

        # Decode and verify token
        payload = jwt.decode(
            token,
            PUBLIC_KEY,
            algorithms=["RS256"],
            audience="target-service",
            options={
                "require": ["exp", "iss", "aud", "scope"]
            }
        )

        # Check issuer
        issuer = payload.get("iss")
        if issuer not in ALLOWED_SERVICES:
            raise HTTPException(
                status_code=403,
                detail=f"Service {issuer} not authorized"
            )

        # Check scope
        scope = payload.get("scope", [])
        if "read" not in scope:
            raise HTTPException(
                status_code=403,
                detail="Missing required scope: read"
            )

        return payload

    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError as e:
        raise HTTPException(status_code=401, detail=f"Invalid token: {e}")

@app.get("/api/data")
async def get_data(payload: dict = Depends(verify_service_token)):
    """Endpoint protected with service authentication"""
    service_name = payload["iss"]
    return {"message": f"Hello from {service_name}"}
```

## Centralized verification

```python
@app.middleware("http")
async def auth_middleware(request: Request, call_next):
    """Centralized authentication for all endpoints"""

    # Skip health checks
    if request.url.path in ["/health", "/live", "/ready"]:
        return await call_next(request)

    # Get authorization header
    auth_header = request.headers.get("Authorization", "")

    if not auth_header.startswith("Bearer "):
        return JSONResponse(
            status_code=401,
            content={"error": "Missing or invalid authorization header"}
        )

    token = auth_header.replace("Bearer ", "", 1)

    try:
        # Verify token
        payload = verify_jwt_token(token)

        # Store service info in request state
        request.state.service_name = payload["iss"]
        request.state.permissions = payload.get("scope", [])

    except Exception as e:
        return JSONResponse(
            status_code=401,
            content={"error": f"Authentication failed: {str(e)}"}
        )

    return await call_next(request)
```

## Monitoring

```
service_auth_requests_total (counter)
service_auth_failures_total (counter)
service_auth_duration_seconds (histogram)
service_auth_token_cache_hits (counter)
service_auth_token_cache_misses (counter)
```

Logging:
```python
logger.info(
    "Service authenticated",
    extra={
        "service_name": payload["iss"],
        "scopes": payload.get("scope", []),
        "client_ip": request.client.host
    }
)
```

## Best practices

✅ **Do**
* Use mTLS for intra-service communication
* Use JWT/OAuth for external API
* Rotate secret keys regularly
* Use short-lived tokens (1 hour)
* Log all authentication events
* Cache verified tokens (for 5-10 minutes)
* Use timing-safe comparison for secrets

❌ **Don't**
* Use API Keys for internal services
* Store secrets in code
* Use long-lived tokens
* Trust tokens without signature verification
* Ignore auth failure metrics
* Log secrets and tokens

## Additional resources

* [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
* [JWT RFC 7519](https://tools.ietf.org/html/rfc7519)
* [mTLS in Kubernetes](https://istio.io/docs/concepts/security/)
* [SPIFFE/SPIRE](https://spiffe.io/)
* [HashiCorp Vault](https://www.vaultproject.io/)
