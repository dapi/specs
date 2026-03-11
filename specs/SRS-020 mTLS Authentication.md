# SRS-020 mTLS Authentication

## Definition

Mutual TLS (mTLS) is a service authentication method where both client and server present X.509 certificates signed by a trusted Certificate Authority (CA), establishing mutual cryptographic proof of identity.

## When to Use

| Scenario | Recommendation |
|----------|----------------|
| Service-to-service within cluster | ✅ Preferred |
| High security requirements | ✅ Required |
| Zero-trust network model | ✅ Required |
| External public APIs | ❌ Use JWT/OAuth instead |
| Legacy systems | ❌ Too complex |

## How It Works

```
Client                          Server
  │                               │
  │──── ClientHello ─────────────>│
  │<─── ServerHello + Cert ───────│
  │──── Client Cert ─────────────>│
  │     (both verify each other)  │
  │<═══════ TLS Session ══════════│
```

1. Server presents its certificate
2. Client verifies server certificate against CA
3. Client presents its certificate
4. Server verifies client certificate against CA
5. Mutual TLS session established

## Configuration

### Nginx

```nginx
server {
    listen 443 ssl;

    ssl_certificate /etc/ssl/certs/server.crt;
    ssl_certificate_key /etc/ssl/private/server.key;

    # CA to verify client certificates
    ssl_client_certificate /etc/ssl/certs/ca.crt;
    ssl_verify_client on;
    ssl_verify_depth 2;

    location / {
        proxy_set_header X-SSL-Client-Serial $ssl_client_serial;
        proxy_set_header X-SSL-Client-DN $ssl_client_s_dn;
        proxy_set_header X-SSL-Client-Verify $ssl_client_verify;
        proxy_pass http://app;
    }
}
```

These identity headers MUST be added only by a trusted TLS-terminating proxy or service mesh sidecar. The application MUST NOT be directly reachable from untrusted networks, and any client-supplied `X-SSL-Client-*` headers MUST be stripped or overwritten at the edge.

### Kubernetes / Istio

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

### Environment Variables

```
SERVICE_AUTH_MTLS_VERIFY=true
SERVICE_AUTH_MTLS_CA_CERT_PATH=/etc/ssl/certs/ca.crt
SERVICE_AUTH_MTLS_CERT_PATH=/etc/ssl/certs/client.crt
SERVICE_AUTH_MTLS_KEY_PATH=/etc/ssl/private/client.key
SERVICE_AUTH_MTLS_VERIFY_DEPTH=2
```

## Implementation

### Client (Python)

```python
import ssl
import httpx

def create_mtls_client(cert_path: str, key_path: str, ca_path: str) -> httpx.Client:
    ssl_context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
    ssl_context.load_cert_chain(cert_path, key_path)
    ssl_context.load_verify_locations(ca_path)
    ssl_context.verify_mode = ssl.CERT_REQUIRED
    return httpx.Client(verify=ssl_context)

client = create_mtls_client(
    cert_path="/etc/ssl/certs/client.crt",
    key_path="/etc/ssl/private/client.key",
    ca_path="/etc/ssl/certs/ca.crt"
)

response = client.get("https://target-service/api/data")
```

### Server-side Certificate Verification (FastAPI)

```python
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()

@app.middleware("http")
async def verify_mtls(request: Request, call_next):
    # Trust these headers only from the local/trusted proxy that terminated mTLS.
    # Block direct access to the application port at the network layer.
    client_verify = request.headers.get("X-SSL-Client-Verify")
    if client_verify != "SUCCESS":
        raise HTTPException(status_code=401, detail="Client certificate required")

    # Extract service name from certificate DN: CN=service-name,O=company
    client_dn = request.headers.get("X-SSL-Client-DN", "")
    request.state.service_name = extract_cn_from_dn(client_dn)

    return await call_next(request)

def extract_cn_from_dn(dn: str) -> str:
    for part in dn.split(","):
        if part.strip().startswith("CN="):
            return part.strip()[3:]
    return ""
```

## Certificate Management

### Certificate Lifecycle

```
Generate CA → Issue Server Cert → Issue Client Cert → Deploy → Monitor Expiry → Rotate
```

### Recommended Certificate TTL

| Environment | Certificate TTL | Rotation Trigger |
|-------------|-----------------|------------------|
| Development | 1 year | Manual |
| Staging | 90 days | At 75% lifetime |
| Production | 30 days | At 75% lifetime |
| High security | 24 hours | Automatic (SPIRE) |

### Monitoring Expiry

```bash
# Check certificate expiry
openssl x509 -in /etc/ssl/certs/client.crt -noout -enddate

# Alert thresholds: 30d, 7d, 1d before expiry
```

### Automated Rotation with SPIFFE/SPIRE

```yaml
agent:
  data_dir: /opt/spire/data/agent
  log_level: INFO
  server_address: spire-server
  server_port: 8081
  trust_domain: example.org
```

## Monitoring

```
service_mtls_handshake_total{result="success|failure"} (counter)
service_mtls_handshake_duration_seconds (histogram)
service_mtls_cert_expiry_seconds{service="name"} (gauge)
service_mtls_verification_failures_total (counter)
```

## Best Practices

✅ **Do**
* Use mTLS for all intra-cluster service communication
* Automate certificate rotation (SPIFFE/SPIRE, cert-manager)
* Alert on certificate expiry at 30/7/1 day thresholds
* Use short-lived certificates (24h–30d) with automatic renewal
* Store private keys in secure storage (Vault, KMS)
* Trust forwarded certificate headers only from a hardened proxy or sidecar

❌ **Don't**
* Use self-signed certs in production without a proper CA
* Store private keys in environment variables or config files
* Set certificate TTL longer than 1 year
* Skip verification of client certificates

## Pros and Cons

**Pros:**
* Highest security level — cryptographic identity, cannot be forged
* No shared secrets — each service has its own key pair
* Mutual verification — both parties authenticate each other
* Works at the transport layer — transparent to application code

**Cons:**
* Certificate management complexity
* TLS handshake overhead (~1–5ms per new connection)
* Requires PKI infrastructure
* Certificate rotation needs coordination

## Additional Resources

* [SPIFFE/SPIRE](https://spiffe.io/)
* [cert-manager for Kubernetes](https://cert-manager.io/)
* [Istio mTLS](https://istio.io/docs/concepts/security/)
* [RFC 8446 — TLS 1.3](https://tools.ietf.org/html/rfc8446)
* [HashiCorp Vault — Secrets Management](https://www.vaultproject.io/)
