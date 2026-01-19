# SRG-006 Audit Logging

Audit Logging is the practice of systematic recording of audit events related to security, access, and important operations in the system for subsequent analysis and incident investigation.

---

## Goals of audit logging

### Security
- Detection of unauthorized access
- Investigation of security incidents
- Compliance with requirements (GDPR, HIPAA, PCI DSS, SOX)
- Forensic analysis

### Operational transparency
- Tracking important operations
- Reconstructing action chains
- Analyzing cause-and-effect relationships
- Proof of operation execution

### Compliance
- Preserving history as required by regulators
- Security audits
- Data storage in accordance with legislation

---

## What to log (Audit Events)

### Authentication and authorization
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "event_type": "user.login",
  "user_id": "user_12345",
  "ip_address": "192.168.1.100",
  "user_agent": "Mozilla/5.0...",
  "result": "success",
  "session_id": "sess_abc123",
  "auth_method": "password"
}
```

Important events:
- `user.login` - successful/unsuccessful login
- `user.logout` - logout
- `session.created` - session creation
- `session.destroyed` - session destruction
- `privilege.escalation` - privilege escalation
- `access.denied` - access denied
- `auth.failed` - failed authentication

### Data access

```json
{
  "timestamp": "2024-01-15T10:35:00Z",
  "event_type": "data.accessed",
  "user_id": "user_12345",
  "resource_type": "user_record",
  "resource_id": "user_67890",
  "action": "read",
  "result": "success",
  "data_classification": "PII",
  "query": "SELECT * FROM users WHERE id = ?"
}
```

Important events:
- `data.accessed` - data access
- `data.created` - record creation
- `data.modified` - data modification
- `data.deleted` - data deletion
- `data.exported` - data export
- `data.shared` - access granting

### User and permission management

```json
{
  "timestamp": "2024-01-15T11:00:00Z",
  "event_type": "user.created",
  "admin_id": "admin_001",
  "target_user_id": "user_99999",
  "target_role": "editor",
  "permissions": ["read", "write"]
}
```

Important events:
- `user.created` - user creation
- `user.modified` - user data modification
- `user.deleted` - user deletion
- `role.assigned` - role assignment
- `permission.granted` - permission granting
- `permission.revoked` - permission revocation
- `group.membership.changed` - group membership changes

### Configuration management

```json
{
  "timestamp": "2024-01-15T11:30:00Z",
  "event_type": "config.changed",
  "user_id": "admin_001",
  "resource": "payment_gateway",
  "changes": {
    "stripe_key": "*** CHANGED ***",
    "webhook_url": "https://api.new.example.com/webhook"
  },
  "previous_version": "config_v45",
  "new_version": "config_v46",
  "reason": "Migration to new payment provider"
}
```

### Financial operations

```json
{
  "timestamp": "2024-01-15T12:00:00Z",
  "event_type": "payment.processed",
  "user_id": "user_12345",
  "amount": 99.99,
  "currency": "USD",
  "payment_method": "card_6789",
  "transaction_id": "txn_abc123def456",
  "status": "completed",
  "compliance_check": "passed"
}
```

### Administrative actions

```json
{
  "timestamp": "2024-01-15T14:00:00Z",
  "event_type": "admin.action",
  "admin_id": "super_admin",
  "action": "database_backup_created",
  "affected_systems": ["primary_db", "replica_db"],
  "duration_ms": 45000
}
```

---

## Audit log format

### Required fields (Minimum Viable)

```json
{
  "timestamp": "2024-01-15T10:30:00Z",           # ISO 8601 with timezone
  "event_type": "user.login",                    # Event type
  "actor_id": "user_12345",                      # Who performed the action
  "result": "success",                           # success | failure
  "correlation_id": "corr_abc123def456"          # For tracing
}
```

### Recommended fields (Standard)

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "event_type": "user.login",
  "actor": {
    "id": "user_12345",
    "type": "user",
    "ip_address": "192.168.1.100",
    "user_agent": "Mozilla/5.0..."
  },
  "action": "login",
  "resource": {
    "type": "user_account",
    "id": "account_67890"
  },
  "result": "success",
  "correlation_id": "corr_abc123def456",
  "session_id": "sess_xyz789",
  "auth_method": "password",
  "compliance_tags": ["GDPR", "SOX"]
}
```

### Full format (Comprehensive)

```json
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "event_type": "payment.processed",
  "version": "1.0",

  "actor": {
    "id": "user_12345",
    "type": "user",
    "identity_provider": "internal",
    "roles": ["customer", "premium"],
    "ip_address": "192.168.1.100",
    "user_agent": "Mozilla/5.0...",
    "location": {
      "country": "US",
      "city": "New York"
    }
  },

  "action": {
    "type": "payment.processed",
    "parameters": {
      "amount": 99.99,
      "currency": "USD",
      "payment_method": "card_6789"
    }
  },

  "resource": {
    "type": "transaction",
    "id": "txn_abc123def456",
    "owner_id": "user_12345",
    "data_classification": "PCI"
  },

  "result": {
    "status": "success",
    "code": 200,
    "message": "Payment processed successfully",
    "reason": null
  },

  "before_state": {
    "user_balance": 150.00
  },

  "after_state": {
    "user_balance": 50.01
  },

  "metadata": {
    "correlation_id": "corr_abc123def456",
    "session_id": "sess_xyz789",
    "request_id": "req_12345",
    "trace_id": "trace_abc123",
    "tags": ["production", "eu-west-1", "v2.1.0"]
  },

  "compliance": {
    "retention_period_days": 2555,
    "regulations": ["PCI-DSS", "GDPR"],
    "auditor_notes": "Standard purchase transaction"
  }
}
```

---

## Logging levels

### Level 0: Minimal
- User login/logout
- Failed authentications
- Permission changes

### Level 1: Standard (recommended)
- All Level 0 events
- Critical data operations
- Financial transactions
- Configuration changes
- Administrative actions

### Level 2: Detailed
- All Level 1 events
- Access to sensitive data (PII, PHI)
- All data changes
- Access attempts (successful and unsuccessful)
- All API calls with parameters

### Level 3: Debug (for forensics)
- All Level 2 events
- All before/after states
- Full request/response (without secrets)
- Network activity
- System calls

---

## Integration in applications

### Django Middleware

```python
class AuditLogMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        actor_id = request.user.id if request.user.is_authenticated else 'anonymous'

        # Log request
        logger.info({
            'event_type': 'request.received',
            'actor_id': actor_id,
            'method': request.method,
            'path': request.path,
            'ip_address': get_client_ip(request)
        })

        response = self.get_response(request)

        # Log response
        logger.info({
            'event_type': 'response.sent',
            'actor_id': actor_id,
            'status_code': response.status_code,
            'content_length': len(response.content)
        })

        return response
```

### Ruby on Rails

```ruby
class AuditLog
  def self.log(current_user, action, resource, result)
    AuditLogEntry.create!(
      timestamp: Time.now.utc,
      user_id: current_user&.id,
      user_ip: RequestContext.ip,
      action: action,
      resource_type: resource.class.name,
      resource_id: resource.id,
      result: result,
      session_id: RequestContext.session_id,
      correlation_id: RequestContext.correlation_id
    )
  end
end

# Usage
class PaymentsController < ApplicationController
  def create
    payment = Payment.new(payment_params)

    if payment.save
      AuditLog.log(current_user, 'payment.created', payment, 'success')
      render json: payment
    else
      AuditLog.log(current_user, 'payment.created', payment, 'failure')
      render json: {errors: payment.errors}, status: :unprocessable_entity
    end
  end
end
```

### Node.js (Express)

```javascript
const auditLogger = (req, res, next) => {
  const startTime = Date.now();
  const actorId = req.user ? req.user.id : 'anonymous';

  // Log incoming request
  logger.info({
    event_type: 'request.start',
    actor_id: actorId,
    method: req.method,
    path: req.path,
    ip_address: req.ip,
    user_agent: req.get('User-Agent')
  });

  // Intercept response
  const originalSend = res.send;
  res.send = function(body) {
    const duration = Date.now() - startTime;

    logger.info({
      event_type: 'response.complete',
      actor_id: actorId,
      method: req.method,
      path: req.path,
      status_code: res.statusCode,
      duration_ms: duration,
      response_size: body.length
    });

    return originalSend.call(this, body);
  };

  next();
};

app.use(auditLogger);
```

---

## Storage configuration

### Elasticsearch

```python
# Index with lifecycle policy
PUT /_index_template/audit_logs
{
  "index_patterns": ["audit-logs-*"],
  "template": {
    "settings": {
      "index.lifecycle.name": "audit_logs_policy",
      "index.lifecycle.rollover_alias": "audit-logs"
    },
    "mappings": {
      "properties": {
        "timestamp": {"type": "date"},
        "event_type": {"type": "keyword"},
        "actor_id": {"type": "keyword"},
        "result": {"type": "keyword"},
        "ip_address": {"type": "ip"},
        "location": {"type": "geo_point"}
      }
    }
  }
}
```

### S3 for long-term storage

```python
import boto3
from datetime import datetime

class S3AuditStorage:
    def __init__(self, bucket):
        self.s3 = boto3.client('s3')
        self.bucket = bucket

    def store_log(self, audit_entry):
        date_str = datetime.utcnow().strftime('%Y/%m/%d')
        key = f"audit-logs/{date_str}/{audit_entry['correlation_id']}.json"

        self.s3.put_object(
            Bucket=self.bucket,
            Key=key,
            Body=json.dumps(audit_entry),
            ServerSideEncryption='AES256',
            StorageClass='GLACIER' if is_old(audit_entry) else 'STANDARD'
        )
```

---

## Analysis and monitoring

### Dashboard in Kibana

```json
{
  "dashboard": {
    "title": "Audit Log Dashboard",
    "panels": [
      {
        "type": "timeseries",
        "title": "Authentications 24h",
        "query": "event_type:user.login"
      },
      {
        "type": "metric",
        "title": "Failed logins",
        "query": "event_type:user.login AND result:failure"
      },
      {
        "type": "table",
        "title": "Critical events",
        "query": "compliance.regulations:*"
      }
    ]
  }
}
```

### Alerts

```yaml
# Elasticsearch Watcher
PUT _watcher/watch/failed_logins
{
  "trigger": {
    "schedule": {"interval": "5m"}
  },
  "input": {
    "search": {
      "request": {
        "indices": ["audit-logs-*"],
        "body": {
          "query": {
            "bool": {
              "must": [
                {"term": {"event_type": "user.login"}},
                {"term": {"result": "failure"}}
              ],
              "filter": {
                "range": {
                  "timestamp": {"gte": "now-5m"}
                }
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {"ctx.payload.hits.total": {"gte": 10}}
  },
  "actions": {
    "send_email": {
      "email": {
        "to": "security@example.com",
        "subject": "⚠️ Multiple failed login attempts detected",
        "body": "More than 10 failed logins in last 5 minutes"
      }
    }
  }
}
```

---

## Audit log security

### Tamper protection

```python
import hashlib
import hmac

class AuditLogSigner:
    def __init__(self, secret_key):
        self.secret_key = secret_key.encode()

    def sign_entry(self, entry):
        """Sign entry to prevent tampering"""
        data = json.dumps(entry, sort_keys=True).encode()
        signature = hmac.new(self.secret_key, data, hashlib.sha256).hexdigest()

        return {
            **entry,
            '_metadata': {
                'signature': signature,
                'algorithm': 'HMAC-SHA256'
            }
        }

    def verify_entry(self, entry):
        """Verify entry signature"""
        if '_metadata' not in entry:
            return False

        signature = entry['_metadata']['signature']
        data = json.dumps({k: v for k, v in entry.items() if k != '_metadata'},
                         sort_keys=True).encode()
        expected = hmac.new(self.secret_key, data, hashlib.sha256).hexdigest()

        return hmac.compare_digest(signature, expected)
```

### Access control

```yaml
# RBAC for audit logs
- role: auditor
  permissions:
    - audit_logs:read
    - audit_logs:search
    - audit_logs:export
  deny:
    - audit_logs:delete
    - audit_logs:modify

- role: security_admin
  permissions:
    - audit_logs:*
  deny:
    - audit_logs:delete  # Nobody can delete
```

---

## Compliance with standards

### GDPR
- Access logs to personal data (Article 30)
- Minimum 6 months retention
- Right to deletion (right to be forgotten)

### HIPAA
- All access to PHI (Protected Health Information)
- Minimum 6 years retention
- Integrity controls

### PCI DSS
- All access to cardholder data
- Minimum 1 year retention, 3 years availability
- Immutable logs

### SOX
- All financial operations
- All system changes
- Minimum 7 years retention
- Cannot be changed/deleted

---

## Incident analysis process

```bash
#!/bin/bash
# analyze-incident.sh

INCIDENT_TIME="2024-01-15T10:30:00"
USER_ID="suspicious_user_123"

echo "=== Incident Analysis ==="
echo "Time: $INCIDENT_TIME"
echo "User: $USER_ID"

echo ""
echo "1. All user actions for 24 hours:"
esearch "actor_id:$USER_ID AND timestamp:[$INCIDENT_TIME-24h TO $INCIDENT_TIME+24h]"

echo ""
echo "2. All logins:"
esearch "actor_id:$USER_ID AND event_type:user.login"

echo ""
echo "3. Critical operations:"
esearch "actor_id:$USER_ID AND (data.classification:PII OR data.classification:PCI)"

echo ""
echo "4. Access permission changes:"
esearch "actor_id:$USER_ID AND event_type:permission.*"

echo ""
echo "Analysis completed. Logs saved in incident_$USER_ID.json"
```

---

*Audit Logging - practice of systematic recording of audit events for security and compliance*
