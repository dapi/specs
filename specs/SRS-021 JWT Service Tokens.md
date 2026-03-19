# SRS-021 JWT Service Tokens

## Definition

JWT (JSON Web Token) Service Tokens are signed tokens used to authenticate service-to-service requests. The token is issued by the calling service, signed with a private key, and verified by the receiving service using the corresponding public key — without requiring a call to an external auth service.

## When to Use

| Scenario | Recommendation |
|----------|----------------|
| External API authentication | ✅ Preferred |
| Inter-service with shared PKI | ✅ Good fit |
| Stateless verification needed | ✅ Good fit |
| Multiple issuers, single receiver | ✅ JWKS mode |
| Immediate token revocation needed | ❌ Use OAuth 2.0 |
| Intra-cluster (same trust domain) | ❌ Use mTLS |

## Token Structure

```
Header.Payload.Signature
```

### Required Claims

```json
{
  "exp": 1710000000,           // Expiration time (Unix timestamp) — always required
  "iat": 1709996400,           // Issued at — always required
  "jti": "uuid-v4-unique-id",  // JWT ID — unique token identifier for audit/replay detection, always required
  "iss": "service-a",          // Issuer — required if SERVICE_AUTH_JWT_ISSUER is configured
  "aud": "target-service",     // Audience — required if SERVICE_AUTH_JWT_AUDIENCE is configured
  "sub": "service-a",          // Subject — recommended
  "scope": ["read", "write"]   // Permissions — optional
}
```

**Validation rules:**

| Claim | Required | Behavior |
|-------|----------|----------|
| `exp` | Always | 401 if expired |
| `iat` | Always | 401 if missing; 401 if issued in the future beyond `SERVICE_AUTH_JWT_CLOCK_SKEW`; 401 if `now - iat > SERVICE_AUTH_JWT_EXPIRATION + SERVICE_AUTH_JWT_CLOCK_SKEW` |
| `jti` | Always | 401 if missing or empty string; if replay detection is enabled, 401 on reused `jti` |
| `iss` | If `SERVICE_AUTH_JWT_ISSUER` env is set | 401 if value doesn't match `SERVICE_AUTH_JWT_ISSUER` |
| `aud` | If `SERVICE_AUTH_JWT_AUDIENCE` env is set (recommended in production) | 401 if value doesn't contain `SERVICE_AUTH_JWT_AUDIENCE` |
| `kid` | Optional | Used to select the matching key from JWKS; if absent, all keys in the set are tried |
| `scope` | Optional | Used for permission checks if present |

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
        "exp": int(time.time()) + 3600,   # 1 hour
        "iat": int(time.time()),
        "jti": str(uuid.uuid4()),
        "scope": scopes
    }
    return jwt.encode(payload, private_key, algorithm="RS256")
```

### Verifying a Service Token

```python
def verify_service_token(token: str, public_key: str,
                         expected_audience: str | None,
                         allowed_issuers: set[str],
                         jwt_expiration: int,
                         jwt_clock_skew: int = 60,
                         replay_cache=None) -> dict:
    try:
        payload = jwt.decode(
            token,
            public_key,
            algorithms=["RS256"],
            audience=expected_audience if expected_audience else None,
            options={
                "require": ["exp", "iat", "jti"],
                "verify_aud": bool(expected_audience),
            }
        )

        now = int(time.time())
        if payload["iat"] > now + jwt_clock_skew:
            raise AuthError("Token issued in the future")

        # iat-based age check — reject even if exp is still valid
        age = now - payload["iat"]
        if age > jwt_expiration + jwt_clock_skew:
            raise AuthError("Token too old")

        jti = payload.get("jti")
        if not jti:
            raise AuthError("Missing jti")

        if allowed_issuers and payload.get("iss") not in allowed_issuers:
            raise AuthError(f"Issuer {payload.get('iss')} not allowed")

        if replay_cache is not None:
            ttl = max(1, int(payload["exp"] - now))
            if not replay_cache.mark_first_seen(jti, ttl=ttl):
                raise AuthError("Replay detected")

        return payload

    except jwt.ExpiredSignatureError:
        raise AuthError("Token expired")
    except jwt.InvalidIssuerError:
        raise AuthError("Invalid issuer")
    except jwt.InvalidTokenError as e:
        raise AuthError(f"Invalid token: {e}")
```

If replay protection is required, `replay_cache` must be backed by a shared store such as Redis with `SETNX`-style semantics and a TTL lasting until `exp`. A bare `jti` claim by itself is not enough to stop token replay.

### Verifying with JWKS

In JWKS mode, the receiver fetches public keys dynamically from the issuer instead of holding a static key file. Keys are cached for 5 minutes.

```python
import time
import threading
import urllib.request
import json

_jwks_cache: dict[str, tuple[list, float]] = {}  # client -> (keys, fetched_at)
_jwks_lock = threading.Lock()
JWKS_CACHE_TTL = 300  # 5 minutes

def _fetch_jwks(client: str, url_template: str) -> list:
    url = url_template.replace("{client}", client)
    with urllib.request.urlopen(url, timeout=5) as resp:
        return json.loads(resp.read())["keys"]

def get_jwks_keys(client: str, url_template: str) -> list:
    with _jwks_lock:
        cached = _jwks_cache.get(client)
        if cached and time.time() - cached[1] < JWKS_CACHE_TTL:
            return cached[0]
        keys = _fetch_jwks(client, url_template)
        _jwks_cache[client] = (keys, time.time())
        return keys

def verify_service_token_jwks(token: str,
                               allowed_clients: set[str],
                               url_template: str,
                               expected_audience: str | None,
                               jwt_expiration: int,
                               jwt_clock_skew: int = 60,
                               replay_cache=None) -> dict:
    header = jwt.get_unverified_header(token)
    issuer = jwt.decode(token, options={"verify_signature": False}).get("iss")

    if issuer not in allowed_clients:
        raise AuthError(f"Issuer {issuer} not in allowed clients")

    keys = get_jwks_keys(issuer, url_template)
    kid = header.get("kid")

    # Filter by kid if present, otherwise try all keys
    candidates = [k for k in keys if not kid or k.get("kid") == kid]
    if not candidates:
        raise AuthError(f"No matching key found for kid={kid}")

    last_error = None
    for jwk in candidates:
        try:
            public_key = jwt.algorithms.RSAAlgorithm.from_jwk(jwk)
            payload = jwt.decode(
                token,
                public_key,
                algorithms=["RS256"],
                audience=expected_audience if expected_audience else None,
                options={
                    "require": ["exp", "iat", "jti"],
                    "verify_aud": bool(expected_audience),
                }
            )
            # Same iat/jti/replay checks as verify_service_token
            now = int(time.time())
            if payload["iat"] > now + jwt_clock_skew:
                raise AuthError("Token issued in the future")
            if now - payload["iat"] > jwt_expiration + jwt_clock_skew:
                raise AuthError("Token too old")
            jti = payload.get("jti")
            if not jti:
                raise AuthError("Missing jti")
            if replay_cache is not None:
                ttl = max(1, int(payload["exp"] - now))
                if not replay_cache.mark_first_seen(jti, ttl=ttl):
                    raise AuthError("Replay detected")
            return payload
        except (jwt.InvalidTokenError, AuthError) as e:
            last_error = e

    raise AuthError(f"Token verification failed: {last_error}")
```

JWKS may contain multiple keys (for key rotation). If `kid` is present, only the matching key is used; otherwise all keys are tried in order.

### JWKS Endpoint (Issuer requirement)

In JWKS mode, the issuer **must** expose `GET /.well-known/jwks.json` returning active public keys in JWK Set format (RFC 7517). During key rotation, the old key must remain in JWKS until all tokens it signed have expired — i.e., at least `SERVICE_AUTH_JWT_EXPIRATION` seconds after the rotation.

Example response:

```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "kid": "2024-01-key",
      "alg": "RS256",
      "n": "...",
      "e": "AQAB"
    }
  ]
}
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
        allowed_issuers=ALLOWED_SERVICES,
        jwt_expiration=3600,
        jwt_clock_skew=60,
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
            allowed_issuers=ALLOWED_SERVICES,
            jwt_expiration=3600,
            jwt_clock_skew=60,
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

**Issuer** (signs tokens, holds the private key):

```
SERVICE_AUTH_JWT_ALGORITHM=RS256
SERVICE_AUTH_JWT_ISSUER=my-service
SERVICE_AUTH_JWT_EXPIRATION=3600
SERVICE_AUTH_JWT_PRIVATE_KEY_PATH=/etc/keys/private.pem
```

**Receiver — Static Public Key mode** (verifies tokens using a local public key file):

```
SERVICE_AUTH_JWT_ALGORITHM=RS256
SERVICE_AUTH_JWT_AUDIENCE=target-service
SERVICE_AUTH_JWT_CLOCK_SKEW=60
SERVICE_AUTH_JWT_PUBLIC_KEY_PATH=/etc/keys/public.pem
```

**Receiver — JWKS mode** (fetches public keys dynamically from the issuer):

```
SERVICE_AUTH_JWT_ALGORITHM=RS256
SERVICE_AUTH_JWT_AUDIENCE=target-service
SERVICE_AUTH_JWT_CLOCK_SKEW=60
SERVICE_AUTH_JWT_JWKS_CLIENTS=service-a,service-b
SERVICE_AUTH_JWT_JWKS_URL_TEMPLATE=https://{client}/.well-known/jwks.json
```

Mode selection rules:
- If `PUBLIC_KEY_PATH` is set → Static Public Key mode
- If `JWKS_CLIENTS` and `JWKS_URL_TEMPLATE` are set → JWKS mode
- If both or neither are set → service must fail at startup with a configuration error

## Monitoring

```
service_jwt_issued_total (counter)
service_jwt_verification_total{result="success|failure"} (counter)
service_jwt_verification_duration_seconds (histogram)
service_jwt_token_cache_hits_total (counter)
service_jwt_token_cache_misses_total (counter)
service_jwt_jwks_fetch_total{client="...", result="success|failure"} (counter)
service_jwt_jwks_cache_hits_total{client="..."} (counter)
```

## Best Practices

✅ **Do**
* Use RS256 (asymmetric) — never HS256 for service tokens
* Set short expiration (1 hour max)
* Include `jti` and back it with a shared cache if replay detection is required
* Cache tokens on the client side until near expiry
* Rotate key pairs regularly (every 90 days)
* Log all verification failures with token metadata (not the token itself)
* Reject tokens whose `iat` is in the future beyond allowed clock skew
* In JWKS mode, include `kid` in the JWT header for efficient key selection
* Keep all active keys in JWKS during rotation (overlap period = token lifetime)

❌ **Don't**
* Use symmetric HMAC (HS256) — requires sharing the secret
* Set expiration longer than 1 hour in production
* Log or expose raw token values
* Trust tokens without verifying the signature
* Skip `aud` (audience) validation in production
* Remove a key from JWKS before all tokens it signed have expired

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
