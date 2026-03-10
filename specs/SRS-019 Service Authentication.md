# SRS-019 Service Authentication


## Definition

Service Authentication is the process of verifying the authenticity of a service making a request to prevent unauthorized access to resources.

## Authentication Types

| Type | Spec | Best For |
|------|------|----------|
| mTLS | [SRS-020](SRS-020%20mTLS%20Authentication.md) | Intra-cluster service-to-service |
| JWT Service Tokens | [SRS-021](SRS-021%20JWT%20Service%20Tokens.md) | External APIs, inter-service |
| OAuth 2.0 Client Credentials | [SRS-022](SRS-022%20OAuth%202.0%20Client%20Credentials.md) | Centralized auth with token management |
| API Keys | [SRS-023](SRS-023%20API%20Keys%20Authentication.md) | Simple integrations, legacy systems |

## Selection Guide

| Scenario | Recommendation |
|----------|----------------|
| Service-to-service within cluster | [mTLS](SRS-020%20mTLS%20Authentication.md) |
| External services | [JWT](SRS-021%20JWT%20Service%20Tokens.md) or [OAuth 2.0](SRS-022%20OAuth%202.0%20Client%20Credentials.md) |
| Simple integrations | [API Keys](SRS-023%20API%20Keys%20Authentication.md) |
| High security requirements | mTLS + JWT |
| Legacy systems | [API Keys](SRS-023%20API%20Keys%20Authentication.md) |

## Common Requirements

All authentication types MUST:
* Verify identity before granting access
* Log all authentication events
* Implement timing-safe secret comparison
* Rotate credentials regularly
* Never store secrets in source code

## Monitoring

```
service_auth_requests_total (counter, labels: type=mtls|jwt|oauth2|apikey)
service_auth_failures_total (counter, labels: type, reason)
service_auth_duration_seconds (histogram, labels: type)
```

## Additional Resources

* [SRS-020 mTLS Authentication](SRS-020%20mTLS%20Authentication.md)
* [SRS-021 JWT Service Tokens](SRS-021%20JWT%20Service%20Tokens.md)
* [SRS-022 OAuth 2.0 Client Credentials](SRS-022%20OAuth%202.0%20Client%20Credentials.md)
* [SRS-023 API Keys Authentication](SRS-023%20API%20Keys%20Authentication.md)
