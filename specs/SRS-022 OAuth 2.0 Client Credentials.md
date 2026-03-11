# SRS-022 OAuth 2.0 Client Credentials

## Definition

OAuth 2.0 Client Credentials is a grant flow (RFC 6749 §4.4) where a service authenticates using its `client_id` and `client_secret` to obtain an access token from a centralized authorization server. The token is then used for subsequent API calls.

## When to Use

| Scenario | Recommendation |
|----------|----------------|
| Centralized token management required | ✅ Preferred |
| Token revocation needed | ✅ Preferred |
| Multiple services, one auth server | ✅ Good fit |
| Scope-based access control | ✅ Good fit |
| No external auth server available | ❌ Use JWT tokens |
| Simple internal services | ❌ Overhead not worth it |

## Flow

```
Service A                    Auth Server              Service B
    │                            │                        │
    │── POST /oauth/token ───────>│                        │
    │   client_id + secret        │                        │
    │<── access_token ────────────│                        │
    │                            │                        │
    │── GET /api/data ────────────────────────────────────>│
    │   Authorization: Bearer <token>                      │
    │                            │                        │
    │                            │<── introspect token ───│
    │                            │─── token valid ────────>│
    │<── response ────────────────────────────────────────│
```

## Obtaining a Token

```bash
curl -X POST https://auth.example.com/oauth/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=service-a" \
  -d "client_secret=secret-key" \
  -d "scope=users:read orders:read"

# Response
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "users:read orders:read"
}
```

## Implementation

### Token Client with Caching

```python
import time
import threading
import httpx
from dataclasses import dataclass
from pathlib import Path

@dataclass
class TokenInfo:
    access_token: str
    expires_at: float
    scope: str

class OAuth2Client:
    def __init__(self, token_url: str, client_id: str, client_secret: str):
        self._token_url = token_url
        self._client_id = client_id
        self._client_secret = client_secret
        self._tokens: dict[str, TokenInfo] = {}
        self._lock = threading.Lock()

    def _scope_cache_key(self, scope: str) -> str:
        # Normalize scope ordering so equivalent requests share the same cache entry.
        return " ".join(sorted(scope.split()))

    def get_token(self, scope: str = "") -> str:
        with self._lock:
            cache_key = self._scope_cache_key(scope)
            # Refresh 60 seconds before expiry
            cached = self._tokens.get(cache_key)
            if cached and cached.expires_at > time.time() + 60:
                return cached.access_token

            token_info = self._fetch_token(scope)
            self._tokens[cache_key] = token_info
            return token_info.access_token

    def _fetch_token(self, scope: str) -> TokenInfo:
        response = httpx.post(
            self._token_url,
            data={
                "grant_type": "client_credentials",
                "client_id": self._client_id,
                "client_secret": self._client_secret,
                "scope": scope,
            },
        )
        response.raise_for_status()
        data = response.json()

        return TokenInfo(
            access_token=data["access_token"],
            expires_at=time.time() + data["expires_in"],
            scope=data.get("scope", ""),
        )

def load_secret(path: str) -> str:
    return Path(path).read_text().strip()

# Usage
auth_client = OAuth2Client(
    token_url="https://auth.example.com/oauth/token",
    client_id="service-a",
    client_secret=load_secret("/etc/secrets/service_client_secret"),
)

def call_service_b():
    token = auth_client.get_token(scope="users:read")
    response = httpx.get(
        "https://service-b/api/users",
        headers={"Authorization": f"Bearer {token}"}
    )
    return response.json()
```

### Token Introspection (Server Side)

```python
async def introspect_token(token: str) -> dict:
    """Validate token via auth server introspection endpoint"""
    response = await httpx.AsyncClient().post(
        "https://auth.example.com/oauth/introspect",
        data={"token": token},
        auth=(RESOURCE_SERVER_ID, RESOURCE_SERVER_SECRET),
    )
    response.raise_for_status()
    data = response.json()

    if not data.get("active"):
        raise AuthError("Token is not active")

    return data

# FastAPI dependency
async def require_token(
    credentials: HTTPAuthorizationCredentials = Security(HTTPBearer())
) -> dict:
    return await introspect_token(credentials.credentials)
```

### Local JWT Verification (Without Introspection)

When the auth server issues JWT access tokens, verification can be done locally:

```python
import jwt

jwks_client = jwt.PyJWKClient("https://auth.example.com/.well-known/jwks.json")

async def verify_access_token(token: str) -> dict:
    """Verify JWT access token using the signing key selected by kid"""
    signing_key = jwks_client.get_signing_key_from_jwt(token)

    payload = jwt.decode(
        token,
        signing_key.key,
        algorithms=["RS256"],
        audience="https://api.example.com",
        options={"require": ["exp", "iss", "aud"]}
    )
    return payload
```

## Configuration

```
SERVICE_AUTH_OAUTH2_TOKEN_URL=https://auth.example.com/oauth/token
SERVICE_AUTH_OAUTH2_CLIENT_ID=service-name
SERVICE_AUTH_OAUTH2_CLIENT_SECRET_PATH=/etc/secrets/service_client_secret
SERVICE_AUTH_OAUTH2_SCOPES=read,write
SERVICE_AUTH_OAUTH2_CACHE_TOKENS=true
SERVICE_AUTH_OAUTH2_TOKEN_CACHE_TTL=3500           # Slightly less than expires_in
SERVICE_AUTH_OAUTH2_INTROSPECT_URL=https://auth.example.com/oauth/introspect
```

## Monitoring

```
service_oauth2_token_requests_total{result="success|failure"} (counter)
service_oauth2_token_request_duration_seconds (histogram)
service_oauth2_token_cache_hits_total (counter)
service_oauth2_token_cache_misses_total (counter)
service_oauth2_introspection_total{result="active|inactive"} (counter)
service_oauth2_introspection_duration_seconds (histogram)
```

## Best Practices

✅ **Do**
* Cache tokens until 60 seconds before expiry
* Load `client_secret` from Vault, KMS, or a mounted secret file
* Use scopes to limit access to only what the service needs
* Prefer local JWT verification over introspection for performance
* Use short `expires_in` (1 hour max)
* Rotate `client_secret` regularly

❌ **Don't**
* Store `client_secret` in source code, checked-in config, or unencrypted files
* Request tokens on every API call — always cache
* Use overly broad scopes
* Skip token expiry validation

## Pros and Cons

**Pros:**
* Standard protocol (RFC 6749) — widely supported
* Centralized token management and revocation
* Scope-based access control
* Can rotate `client_secret` without service restart (with caching)

**Cons:**
* Additional HTTP request required to obtain token (mitigated by caching)
* Requires auth server infrastructure
* Higher latency if using introspection on every request

## Additional Resources

* [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
* [Token Introspection RFC 7662](https://tools.ietf.org/html/rfc7662)
* [OAuth 2.0 for Machine-to-Machine](https://auth0.com/docs/get-started/authentication-and-authorization-flow/client-credentials-flow)
