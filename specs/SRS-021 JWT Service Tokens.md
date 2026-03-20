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

The issuer builds a payload with `iss`, `sub`, `aud`, `exp`, `iat`, `jti`, and optionally `scope`, then signs it with RS256 using the service's private key. The `jti` must be a UUID v4 or equivalent globally unique value.

### Verifying a Service Token (Static Public Key mode)

The receiver decodes and verifies the token using the issuer's public key:

1. Verify RS256 signature against the configured public key.
2. Reject if `exp` is in the past.
3. Reject if `iat` is in the future beyond `SERVICE_AUTH_JWT_CLOCK_SKEW`.
4. Reject if `now - iat > SERVICE_AUTH_JWT_EXPIRATION + SERVICE_AUTH_JWT_CLOCK_SKEW` (age check independent of `exp`).
5. Reject if `jti` is missing or empty.
6. Reject if `iss` is not in the allowed issuers set.
7. If replay detection is enabled: reject if `jti` has been seen before; otherwise mark it as seen with TTL = `exp - now`.

If replay protection is required, the seen-jti store must be a shared store (e.g., Redis with `SETNX`-style semantics) with TTL lasting until `exp`. A bare `jti` claim by itself is not enough to stop token replay.

### JWKS Flow

```
Issuer                     Receiver                  Issuer JWKS
  |                            |                          |
  |--- POST /api (JWT) ------->|                          |
  |                            |-- read iss from header   |
  |                            |                          |
  |                            |  [cache miss]            |
  |                            |--- GET /.well-known/ --->|
  |                            |       jwks.json          |
  |                            |<-- { keys: [...] } ------|
  |                            |-- store in memory cache  |
  |                            |   (TTL = 5 min)          |
  |                            |                          |
  |                            |  [cache hit]             |
  |                            |-- read from cache        |
  |                            |                          |
  |                            |-- verify signature       |
  |                            |-- validate claims        |
  |<-- 200 OK / 401 Unauth. ---|                          |
```

### Verifying with JWKS

In JWKS mode, the receiver fetches public keys dynamically from the issuer instead of holding a static key file. Keys are cached in memory for 5 minutes (fixed TTL, not configurable).

Verification steps:

1. Extract `iss` from the token (without verifying the signature).
2. Reject if `iss` is not in `SERVICE_AUTH_JWT_JWKS_CLIENTS`.
3. Fetch the JWKS from `SERVICE_AUTH_JWT_JWKS_URL_TEMPLATE` (substituting `{client}` with `iss`), or return the cached result if still fresh.
4. If `kid` is present in the token header, select only the key with the matching `kid`; otherwise try all keys in the set.
5. Attempt signature verification with each candidate key in order. Stop at first success.
6. Apply the same `iat`/`exp`/`jti`/replay checks as in Static Public Key mode.
7. Return the verified payload, or 401 if no candidate key succeeds.

JWKS may contain multiple keys to support key rotation. If `kid` is specified, only the matching key is tried; otherwise all keys are attempted.

### JWKS Endpoint (Issuer requirement)

In JWKS mode, the issuer **must** expose `GET /.well-known/jwks.json` returning active public keys in JWK Set format (RFC 7517). During key rotation, the old key must remain in JWKS until all tokens it signed have expired — i.e., at least `SERVICE_AUTH_JWT_EXPIRATION` seconds after the rotation.

### Token Caching (Issuer side)

To avoid creating a new token on every outgoing request, the issuer should cache tokens and reuse them until shortly before expiry (e.g., refresh 60 seconds before `exp`). The cache is keyed by target service name and held in process memory.

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

## Recommended Libraries

### Python

| Role | Library | Notes |
|------|---------|-------|
| Issuer + Receiver | `PyJWT` + `cryptography` | RS256, JWK support via `jwt.algorithms.RSAAlgorithm.from_jwk()` |
| Receiver (JWKS) | `python-jose` | Built-in JWKS fetch and caching |

### Node.js

| Role | Library | Notes |
|------|---------|-------|
| Issuer + Receiver | `jsonwebtoken` | De facto standard, RS256, no built-in JWKS |
| Receiver (JWKS) | `jwks-rsa` | JWKS fetch and caching, integrates with `jsonwebtoken` |
| Receiver (JWKS) | `jose` | Full JWKS support, modern API, ESM/CJS |

### Go

| Role | Library | Notes |
|------|---------|-------|
| Issuer + Receiver | `golang-jwt/jwt` | Go standard, RS256 |
| Receiver (JWKS) | `MicahParks/keyfunc` | JWKS fetch and caching, integrates with `golang-jwt/jwt` |

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
