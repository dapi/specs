# SRS-021 JWT Service Tokens

**Status**: APPROVED
**Related**: [SRS-019 Service Authentication](SRS-019%20Service%20Authentication.md), [SRS-018 Secrets Management](SRS-018%20Secrets%20Management.md)

## Definition

JWT (JSON Web Token) Service Tokens are signed tokens used to authenticate service-to-service requests. The token is issued by the calling service, signed with a private key, and verified by the receiving service using the corresponding public key — without requiring a call to an external auth service.

## When to Use

| Scenario | Recommendation |
|----------|----------------|
| External API authentication | ✅ Preferred |
| Inter-service with shared PKI | ✅ Good fit |
| Stateless verification needed | ✅ Good fit |
| Immediate token revocation needed | ❌ Use OAuth 2.0 |
| Intra-cluster (same trust domain) | ❌ Use mTLS |

## Token Structure

```
Header.Payload.Signature
```

### Required Claims

```json
{
  "iss": "service-a",          // Issuer — calling service name
  "sub": "service-a",          // Subject — same as issuer for service tokens
  "aud": "target-service",     // Audience — intended recipient
  "exp": 1710000000,           // Expiration time (Unix timestamp)
  "iat": 1709996400,           // Issued at
  "jti": "uuid-v4-unique-id",  // JWT ID — for replay prevention
  "scope": ["read", "write"]   // Permissions
}
```

## Implementation

### Creating a Service Token

```python
import jwt
import time
import uuid

def create_service_token(service_name: str, target_service: str,
                         private_key: str, scopes: list[str]) -> str:
    payload = {
        "iss": service_name,
        "sub": service_name,
        "aud": target_service,
        "exp": time.time() + 3600,   # 1 hour
        "iat": time.time(),
        "jti": str(uuid.uuid4()),
        "scope": scopes
    }
    return jwt.encode(payload, private_key, algorithm="RS256")
```

### Verifying a Service Token

```python
def verify_service_token(token: str, public_key: str,
                         expected_audience: str,
                         allowed_issuers: list[str]) -> dict:
    try:
        payload = jwt.decode(
            token,
            public_key,
            algorithms=["RS256"],
            audience=expected_audience,
            options={"require": ["exp", "iss", "aud", "jti", "scope"]}
        )

        if payload["iss"] not in allowed_issuers:
            raise ValueError(f"Issuer {payload['iss']} not in allowed list")

        return payload

    except jwt.ExpiredSignatureError:
        raise AuthError("Token expired")
    except jwt.InvalidIssuerError:
        raise AuthError("Invalid issuer")
    except jwt.InvalidTokenError as e:
        raise AuthError(f"Invalid token: {e}")
```

### FastAPI Middleware

```python
from fastapi import FastAPI, Depends, HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

app = FastAPI()
security = HTTPBearer()

PUBLIC_KEY = open("/etc/keys/public.pem").read()
ALLOWED_SERVICES = {"service-a", "service-b", "service-c"}

def get_service(
    credentials: HTTPAuthorizationCredentials = Security(security)
) -> dict:
    payload = verify_service_token(
        token=credentials.credentials,
        public_key=PUBLIC_KEY,
        expected_audience="target-service",
        allowed_issuers=list(ALLOWED_SERVICES)
    )
    return payload

@app.get("/api/data")
async def get_data(service: dict = Depends(get_service)):
    return {"message": f"Hello from {service['iss']}"}
```

### Centralized Auth Middleware

```python
@app.middleware("http")
async def auth_middleware(request: Request, call_next):
    if request.url.path in ["/health", "/live", "/ready"]:
        return await call_next(request)

    auth_header = request.headers.get("Authorization", "")
    if not auth_header.startswith("Bearer "):
        return JSONResponse(status_code=401,
                            content={"error": "Missing authorization header"})

    try:
        payload = verify_service_token(
            token=auth_header.removeprefix("Bearer "),
            public_key=PUBLIC_KEY,
            expected_audience="this-service",
            allowed_issuers=ALLOWED_SERVICES
        )
        request.state.service_name = payload["iss"]
        request.state.scopes = payload.get("scope", [])
    except AuthError as e:
        return JSONResponse(status_code=401, content={"error": str(e)})

    return await call_next(request)
```

## Token Caching

To avoid creating a new token on every request, cache tokens until just before expiry:

```python
import threading
from dataclasses import dataclass

@dataclass
class CachedToken:
    token: str
    expires_at: float

_token_cache: dict[str, CachedToken] = {}
_lock = threading.Lock()

def get_service_token(target_service: str) -> str:
    with _lock:
        cached = _token_cache.get(target_service)
        # Refresh 60 seconds before expiry
        if cached and cached.expires_at > time.time() + 60:
            return cached.token

        token = create_service_token(
            service_name=MY_SERVICE_NAME,
            target_service=target_service,
            private_key=PRIVATE_KEY,
            scopes=["read"]
        )
        _token_cache[target_service] = CachedToken(
            token=token,
            expires_at=time.time() + 3600
        )
        return token
```

## Configuration

```
SERVICE_AUTH_JWT_ALGORITHM=RS256
SERVICE_AUTH_JWT_ISSUER=my-service
SERVICE_AUTH_JWT_AUDIENCE=target-service
SERVICE_AUTH_JWT_EXPIRATION=3600
SERVICE_AUTH_JWT_CLOCK_SKEW=60
SERVICE_AUTH_JWT_PRIVATE_KEY_PATH=/etc/keys/private.pem
SERVICE_AUTH_JWT_PUBLIC_KEY_PATH=/etc/keys/public.pem
```

## Monitoring

```
service_jwt_issued_total (counter)
service_jwt_verification_total{result="success|failure"} (counter)
service_jwt_verification_duration_seconds (histogram)
service_jwt_token_cache_hits_total (counter)
service_jwt_token_cache_misses_total (counter)
```

## Best Practices

✅ **Do**
* Use RS256 (asymmetric) — never HS256 for service tokens
* Set short expiration (1 hour max)
* Include `jti` claim for replay prevention
* Cache tokens on the client side until near expiry
* Rotate key pairs regularly (every 90 days)
* Log all verification failures with token metadata (not the token itself)

❌ **Don't**
* Use symmetric HMAC (HS256) — requires sharing the secret
* Set expiration longer than 24 hours
* Log or expose raw token values
* Trust tokens without verifying the signature
* Skip `aud` (audience) validation

## Pros and Cons

**Pros:**
* Stateless verification — no auth service call needed
* Easy to use — standard HTTP `Authorization: Bearer` header
* Built-in expiration
* Supported by virtually every framework

**Cons:**
* Cannot revoke immediately without a blacklist
* Requires clock synchronization (NTP)
* Token size ~1–2KB per request

## Additional Resources

* [JWT RFC 7519](https://tools.ietf.org/html/rfc7519)
* [JWK RFC 7517](https://tools.ietf.org/html/rfc7517)
* [jwt.io debugger](https://jwt.io/)
* [PyJWT library](https://pyjwt.readthedocs.io/)
