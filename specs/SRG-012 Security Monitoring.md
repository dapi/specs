# SRG-012 Security Monitoring

| Field | Value |
|-------|-------|
| **Status** | APPROVED |
| **Version** | 1.0 |
| **Priority** | P1 |
| **Complexity** | High |
| **Target Audience** | Security Engineers, SRE, DevOps, Architects |

## 1. Introduction

### 1.1 Purpose

This specification defines standards for security monitoring in distributed systems. Security monitoring provides visibility into security-relevant events, enables threat detection, and supports incident response and compliance requirements.

### 1.2 Scope

This specification covers:
- Security observability architecture
- SIEM integration patterns
- Security logging and audit trails
- Threat detection and anomaly detection
- Runtime security monitoring
- Compliance monitoring
- Incident response integration

### 1.3 Related Specifications

- [SRG-002 Logging](SRG-002%20Logging.md) - General logging standards
- [SRG-003 Error Tracking](SRG-003%20Error%20Tracking.md) - Error monitoring
- [SRG-005 Alerting Rules](SRG-005%20Alerting%20Rules.md) - Alert management
- [SRO-004 Multi-Region DR](SRO-004%20Multi-Region%20DR.md) - Disaster recovery

## 2. Security Observability Architecture

### 2.1 Security Monitoring Layers

```
┌─────────────────────────────────────────────────────────────────┐
│                    Security Operations Center                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │    SIEM      │  │   SOAR       │  │  Threat Intelligence │  │
│  │  (Splunk,    │  │  (Automated  │  │   (Feeds, IOCs)      │  │
│  │   Elastic)   │  │   Response)  │  │                      │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
└─────────┼─────────────────┼─────────────────────┼──────────────┘
          │                 │                     │
          ▼                 ▼                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Security Data Lake                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Security     │  │  Audit       │  │  Network Flow        │  │
│  │ Events       │  │  Logs        │  │  Data                │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
          ▲                 ▲                     ▲
          │                 │                     │
┌─────────┴─────────────────┴─────────────────────┴──────────────┐
│                    Collection Layer                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Host-based   │  │  Network     │  │  Application         │  │
│  │ Agents       │  │  Sensors     │  │  Instrumentation     │  │
│  │ (Falco,      │  │  (Zeek,      │  │  (OWASP, WAF)        │  │
│  │  OSSEC)      │  │   Suricata)  │  │                      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
          ▲                 ▲                     ▲
          │                 │                     │
┌─────────┴─────────────────┴─────────────────────┴──────────────┐
│                    Infrastructure                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Kubernetes   │  │  Cloud       │  │  Applications        │  │
│  │ Nodes        │  │  Services    │  │  & Services          │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Defense in Depth Monitoring

| Layer | Monitoring Focus | Tools |
|-------|-----------------|-------|
| **Perimeter** | DDoS, WAF events, Firewall logs | Cloudflare, AWS WAF, ModSecurity |
| **Network** | Traffic analysis, IDS/IPS | Zeek, Suricata, VPC Flow Logs |
| **Host** | System calls, File integrity | Falco, OSSEC, Wazuh |
| **Container** | Runtime behavior, Image scanning | Falco, Trivy, Aqua |
| **Application** | Auth events, API abuse | Custom logging, OWASP ZAP |
| **Data** | Access patterns, Encryption | Database audit, KMS logs |

## 3. Security Logging Standards

### 3.1 Security Event Categories

```yaml
# Security Event Taxonomy
security_events:
  authentication:
    - login_success
    - login_failure
    - logout
    - session_timeout
    - mfa_challenge
    - mfa_success
    - mfa_failure
    - password_change
    - password_reset

  authorization:
    - access_granted
    - access_denied
    - privilege_escalation
    - role_change
    - permission_change

  data_access:
    - data_read
    - data_write
    - data_delete
    - data_export
    - bulk_operation

  system:
    - service_start
    - service_stop
    - config_change
    - secret_access
    - certificate_operation

  network:
    - connection_established
    - connection_blocked
    - dns_query
    - tls_error

  threat:
    - malware_detected
    - intrusion_attempt
    - anomaly_detected
    - policy_violation
```

### 3.2 Security Log Format

```json
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "event_type": "authentication",
  "event_name": "login_failure",
  "severity": "warning",
  "source": {
    "service": "auth-service",
    "instance": "auth-service-7b9f5d4c8-x2k4m",
    "version": "1.2.3"
  },
  "actor": {
    "user_id": "user-123",
    "username": "john.doe@example.com",
    "session_id": "sess-abc123",
    "ip_address": "192.168.1.100",
    "user_agent": "Mozilla/5.0...",
    "geo_location": {
      "country": "US",
      "city": "New York",
      "coordinates": [40.7128, -74.0060]
    }
  },
  "target": {
    "resource_type": "user_account",
    "resource_id": "user-123",
    "action": "authenticate"
  },
  "outcome": {
    "status": "failure",
    "reason": "invalid_password",
    "attempt_count": 3
  },
  "context": {
    "request_id": "req-xyz789",
    "trace_id": "trace-abc123",
    "client_fingerprint": "fp-12345"
  },
  "risk_score": 75,
  "tags": ["brute_force_candidate", "unusual_location"]
}
```

### 3.3 Audit Trail Implementation

```go
package audit

import (
    "context"
    "encoding/json"
    "time"
)

type AuditEvent struct {
    ID            string                 `json:"id"`
    Timestamp     time.Time              `json:"timestamp"`
    EventType     string                 `json:"event_type"`
    EventName     string                 `json:"event_name"`
    Severity      string                 `json:"severity"`
    Actor         Actor                  `json:"actor"`
    Target        Target                 `json:"target"`
    Outcome       Outcome                `json:"outcome"`
    Changes       []Change               `json:"changes,omitempty"`
    Context       map[string]interface{} `json:"context"`
    Metadata      map[string]interface{} `json:"metadata"`
}

type Actor struct {
    UserID      string            `json:"user_id"`
    Username    string            `json:"username"`
    Role        string            `json:"role"`
    IPAddress   string            `json:"ip_address"`
    UserAgent   string            `json:"user_agent"`
    SessionID   string            `json:"session_id"`
    ServiceName string            `json:"service_name,omitempty"`
    Attributes  map[string]string `json:"attributes,omitempty"`
}

type Target struct {
    ResourceType string `json:"resource_type"`
    ResourceID   string `json:"resource_id"`
    ResourceName string `json:"resource_name,omitempty"`
    Action       string `json:"action"`
}

type Outcome struct {
    Status     string `json:"status"` // success, failure, partial
    Reason     string `json:"reason,omitempty"`
    ErrorCode  string `json:"error_code,omitempty"`
}

type Change struct {
    Field    string      `json:"field"`
    OldValue interface{} `json:"old_value"`
    NewValue interface{} `json:"new_value"`
}

type AuditLogger interface {
    Log(ctx context.Context, event AuditEvent) error
    Query(ctx context.Context, filter AuditFilter) ([]AuditEvent, error)
}

type AuditFilter struct {
    StartTime    time.Time
    EndTime      time.Time
    EventTypes   []string
    ActorUserID  string
    ResourceType string
    ResourceID   string
    Severity     []string
    Limit        int
    Offset       int
}

// ImmutableAuditLogger ensures audit logs cannot be modified
type ImmutableAuditLogger struct {
    writer    AuditWriter
    hasher    HashChain
    encryptor Encryptor
}

func (l *ImmutableAuditLogger) Log(ctx context.Context, event AuditEvent) error {
    // Add timestamp and unique ID
    event.ID = generateUUID()
    event.Timestamp = time.Now().UTC()

    // Serialize event
    data, err := json.Marshal(event)
    if err != nil {
        return err
    }

    // Add to hash chain for integrity
    hash := l.hasher.AddBlock(data)

    // Encrypt sensitive fields
    encryptedData, err := l.encryptor.Encrypt(data)
    if err != nil {
        return err
    }

    // Write to immutable storage
    return l.writer.Write(ctx, AuditRecord{
        Data:     encryptedData,
        Hash:     hash,
        PrevHash: l.hasher.PreviousHash(),
    })
}

// HashChain provides cryptographic integrity verification
type HashChain struct {
    prevHash []byte
}

func (h *HashChain) AddBlock(data []byte) []byte {
    hash := sha256.Sum256(append(h.prevHash, data...))
    h.prevHash = hash[:]
    return hash[:]
}

func (h *HashChain) Verify(records []AuditRecord) bool {
    var prevHash []byte
    for _, record := range records {
        expectedHash := sha256.Sum256(append(prevHash, record.Data...))
        if !bytes.Equal(expectedHash[:], record.Hash) {
            return false
        }
        prevHash = record.Hash
    }
    return true
}
```

### 3.4 Security Event Enrichment

```python
from dataclasses import dataclass
from typing import Optional, Dict, List
import geoip2.database
import hashlib

@dataclass
class EnrichedSecurityEvent:
    original_event: Dict
    geo_data: Optional[Dict]
    threat_intel: Optional[Dict]
    user_context: Optional[Dict]
    risk_score: int
    tags: List[str]

class SecurityEventEnricher:
    def __init__(
        self,
        geoip_db_path: str,
        threat_intel_client,
        user_service_client
    ):
        self.geoip_reader = geoip2.database.Reader(geoip_db_path)
        self.threat_intel = threat_intel_client
        self.user_service = user_service_client

    async def enrich(self, event: Dict) -> EnrichedSecurityEvent:
        """Enrich security event with additional context"""

        tags = []
        risk_score = 0

        # Geo enrichment
        geo_data = None
        ip_address = event.get('actor', {}).get('ip_address')
        if ip_address:
            geo_data = self._enrich_geo(ip_address)
            if geo_data and geo_data.get('is_anonymous_proxy'):
                tags.append('anonymous_proxy')
                risk_score += 30

        # Threat intelligence enrichment
        threat_intel = await self._enrich_threat_intel(event)
        if threat_intel:
            if threat_intel.get('is_known_bad'):
                tags.append('known_threat_actor')
                risk_score += 50
            if threat_intel.get('ioc_matches'):
                tags.extend(threat_intel['ioc_matches'])
                risk_score += 40

        # User context enrichment
        user_context = None
        user_id = event.get('actor', {}).get('user_id')
        if user_id:
            user_context = await self._enrich_user_context(user_id, event)
            if user_context:
                if user_context.get('unusual_location'):
                    tags.append('unusual_location')
                    risk_score += 20
                if user_context.get('unusual_time'):
                    tags.append('unusual_time')
                    risk_score += 15
                if user_context.get('new_device'):
                    tags.append('new_device')
                    risk_score += 10

        # Event-specific risk scoring
        event_risk = self._calculate_event_risk(event)
        risk_score += event_risk

        return EnrichedSecurityEvent(
            original_event=event,
            geo_data=geo_data,
            threat_intel=threat_intel,
            user_context=user_context,
            risk_score=min(risk_score, 100),
            tags=tags
        )

    def _enrich_geo(self, ip_address: str) -> Optional[Dict]:
        """Enrich with geolocation data"""
        try:
            response = self.geoip_reader.city(ip_address)
            return {
                'country': response.country.iso_code,
                'city': response.city.name,
                'latitude': response.location.latitude,
                'longitude': response.location.longitude,
                'is_anonymous_proxy': response.traits.is_anonymous_proxy,
                'is_tor_exit_node': response.traits.is_tor_exit_node
            }
        except Exception:
            return None

    async def _enrich_threat_intel(self, event: Dict) -> Optional[Dict]:
        """Check threat intelligence feeds"""
        ip_address = event.get('actor', {}).get('ip_address')
        if not ip_address:
            return None

        # Check multiple threat intel sources
        results = await self.threat_intel.check_ip(ip_address)

        return {
            'is_known_bad': results.get('is_malicious', False),
            'threat_type': results.get('threat_type'),
            'confidence': results.get('confidence'),
            'ioc_matches': results.get('matched_indicators', []),
            'first_seen': results.get('first_seen'),
            'last_seen': results.get('last_seen')
        }

    async def _enrich_user_context(
        self,
        user_id: str,
        event: Dict
    ) -> Optional[Dict]:
        """Add user behavioral context"""
        user_profile = await self.user_service.get_security_profile(user_id)
        if not user_profile:
            return None

        event_location = event.get('actor', {}).get('geo_location', {})
        event_time = event.get('timestamp')
        device_fp = event.get('context', {}).get('client_fingerprint')

        return {
            'unusual_location': not self._is_known_location(
                event_location,
                user_profile.get('known_locations', [])
            ),
            'unusual_time': not self._is_normal_time(
                event_time,
                user_profile.get('active_hours')
            ),
            'new_device': device_fp not in user_profile.get('known_devices', []),
            'risk_level': user_profile.get('risk_level', 'normal'),
            'recent_security_events': user_profile.get('recent_events_count', 0)
        }

    def _calculate_event_risk(self, event: Dict) -> int:
        """Calculate risk score based on event type"""
        risk_weights = {
            ('authentication', 'login_failure'): 10,
            ('authentication', 'mfa_failure'): 15,
            ('authorization', 'access_denied'): 10,
            ('authorization', 'privilege_escalation'): 40,
            ('data_access', 'bulk_operation'): 25,
            ('data_access', 'data_export'): 20,
            ('system', 'config_change'): 15,
            ('system', 'secret_access'): 20,
            ('threat', 'malware_detected'): 50,
            ('threat', 'intrusion_attempt'): 45,
        }

        event_key = (event.get('event_type'), event.get('event_name'))
        return risk_weights.get(event_key, 5)
```

## 4. SIEM Integration

### 4.1 SIEM Architecture

```yaml
# Elastic SIEM Configuration
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: security-siem
spec:
  version: 8.11.0
  nodeSets:
  - name: hot
    count: 3
    config:
      node.roles: ["master", "data_hot", "ingest"]
      xpack.security.enabled: true
      xpack.security.transport.ssl.enabled: true
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 8Gi
              cpu: 2
            limits:
              memory: 16Gi
              cpu: 4
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 500Gi
        storageClassName: fast-ssd
  - name: warm
    count: 2
    config:
      node.roles: ["data_warm"]
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 2Ti
        storageClassName: standard
---
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: security-siem
spec:
  version: 8.11.0
  count: 2
  elasticsearchRef:
    name: security-siem
  config:
    xpack.security.enabled: true
    xpack.encryptedSavedObjects.encryptionKey: "${KIBANA_ENCRYPTION_KEY}"
```

### 4.2 Log Shipping Configuration

```yaml
# Filebeat for security log collection
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: security-filebeat
spec:
  type: filebeat
  version: 8.11.0
  elasticsearchRef:
    name: security-siem
  config:
    filebeat.autodiscover:
      providers:
      - type: kubernetes
        node: ${NODE_NAME}
        hints.enabled: true
        hints.default_config:
          type: container
          paths:
            - /var/log/containers/*${data.kubernetes.container.id}.log

    filebeat.inputs:
    - type: log
      enabled: true
      paths:
        - /var/log/audit/audit.log
      tags: ["linux_audit"]
      processors:
        - decode_json_fields:
            fields: ["message"]
            target: "audit"

    - type: log
      enabled: true
      paths:
        - /var/log/falco/*.log
      tags: ["falco"]
      processors:
        - decode_json_fields:
            fields: ["message"]
            target: "falco"

    processors:
    - add_kubernetes_metadata:
        host: ${NODE_NAME}
        matchers:
        - logs_path:
            logs_path: "/var/log/containers/"
    - add_cloud_metadata: ~
    - add_host_metadata: ~

    output.elasticsearch:
      hosts: ["https://security-siem-es-http:9200"]
      protocol: https
      ssl.certificate_authorities:
        - /etc/certificate/ca.crt
      index: "security-logs-%{+yyyy.MM.dd}"

    setup.ilm:
      enabled: true
      rollover_alias: "security-logs"
      pattern: "{now/d}-000001"
      policy_name: "security-logs-policy"
```

### 4.3 Detection Rules

```yaml
# Elastic Security Detection Rules
# Rule: Brute Force Authentication Attempt
---
name: "Brute Force Authentication Attempt"
description: "Detects multiple failed login attempts from the same source"
risk_score: 73
severity: high
type: threshold
index:
  - security-logs-*
language: kuery
query: |
  event_type:authentication AND event_name:login_failure
threshold:
  field:
    - actor.ip_address
    - actor.username
  value: 5
  cardinality:
    field: actor.username
    value: 1
from: now-5m
interval: 1m
actions:
  - action_type: .slack
    params:
      message: |
        🚨 Brute Force Attack Detected
        Source IP: {{context.rule.threshold_field_0}}
        Target User: {{context.rule.threshold_field_1}}
        Attempts: {{state.signals_count}}
tags:
  - authentication
  - brute_force
  - attack

---
# Rule: Privilege Escalation Attempt
name: "Privilege Escalation Attempt"
description: "Detects unauthorized privilege escalation"
risk_score: 85
severity: critical
type: eql
index:
  - security-logs-*
language: eql
query: |
  sequence by actor.user_id with maxspan=1h
    [authorization where event_name == "role_change"]
    [data_access where event_name == "bulk_operation" or event_name == "data_export"]
actions:
  - action_type: .pagerduty
    params:
      severity: critical
      summary: "Privilege escalation followed by data access detected"

---
# Rule: Anomalous Data Export
name: "Anomalous Data Export Volume"
description: "Detects unusually large data exports"
risk_score: 65
severity: medium
type: machine_learning
machine_learning_job_id: security_data_export_anomaly
anomaly_threshold: 75
```

## 5. Runtime Security Monitoring

### 5.1 Falco Configuration

```yaml
# Falco DaemonSet for Kubernetes runtime security
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: security
spec:
  selector:
    matchLabels:
      app: falco
  template:
    metadata:
      labels:
        app: falco
    spec:
      serviceAccountName: falco
      hostNetwork: true
      hostPID: true
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      containers:
      - name: falco
        image: falcosecurity/falco:0.36.2
        securityContext:
          privileged: true
        args:
          - /usr/bin/falco
          - --cri=/run/containerd/containerd.sock
          - -K=/var/run/secrets/kubernetes.io/serviceaccount/token
          - -k=https://kubernetes.default
          - --k8s-node=${FALCO_K8S_NODE}
          - -pk
        env:
        - name: FALCO_K8S_NODE
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - mountPath: /host/var/run/docker.sock
          name: docker-socket
          readOnly: true
        - mountPath: /run/containerd/containerd.sock
          name: containerd-socket
          readOnly: true
        - mountPath: /host/dev
          name: dev-fs
          readOnly: true
        - mountPath: /host/proc
          name: proc-fs
          readOnly: true
        - mountPath: /etc/falco
          name: falco-config
      volumes:
      - name: docker-socket
        hostPath:
          path: /var/run/docker.sock
      - name: containerd-socket
        hostPath:
          path: /run/containerd/containerd.sock
      - name: dev-fs
        hostPath:
          path: /dev
      - name: proc-fs
        hostPath:
          path: /proc
      - name: falco-config
        configMap:
          name: falco-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-config
  namespace: security
data:
  falco.yaml: |
    rules_file:
      - /etc/falco/falco_rules.yaml
      - /etc/falco/falco_rules.local.yaml
      - /etc/falco/k8s_audit_rules.yaml
      - /etc/falco/custom_rules.yaml

    json_output: true
    json_include_output_property: true

    log_stderr: true
    log_syslog: false
    log_level: info

    priority: debug

    buffered_outputs: false

    outputs:
      rate: 1
      max_burst: 1000

    stdout_output:
      enabled: true

    http_output:
      enabled: true
      url: "http://falco-sidekick:2801/"

    grpc:
      enabled: true
      bind_address: "0.0.0.0:5060"
      threadiness: 8

    grpc_output:
      enabled: true

  custom_rules.yaml: |
    # Custom Falco Rules

    - rule: Sensitive File Access
      desc: Detect access to sensitive files
      condition: >
        open_read and
        (fd.name startswith /etc/shadow or
         fd.name startswith /etc/passwd or
         fd.name startswith /root/.ssh/ or
         fd.name contains id_rsa or
         fd.name contains .pem) and
        not proc.name in (sshd, sudo, su, passwd)
      output: >
        Sensitive file accessed
        (user=%user.name command=%proc.cmdline file=%fd.name container=%container.id
        image=%container.image.repository k8s.pod=%k8s.pod.name k8s.ns=%k8s.ns.name)
      priority: WARNING
      tags: [filesystem, sensitive]

    - rule: Container Shell Spawned
      desc: Detect shell spawned in container
      condition: >
        spawned_process and
        container and
        proc.name in (bash, sh, zsh, ash, dash) and
        proc.pname != entrypoint.sh and
        not container.image.repository in (
          docker.io/library/busybox,
          gcr.io/distroless/debug
        )
      output: >
        Shell spawned in container
        (user=%user.name shell=%proc.name parent=%proc.pname
        container=%container.id image=%container.image.repository
        k8s.pod=%k8s.pod.name k8s.ns=%k8s.ns.name)
      priority: NOTICE
      tags: [container, shell]

    - rule: Crypto Mining Activity
      desc: Detect potential crypto mining
      condition: >
        spawned_process and
        (proc.name in (xmrig, minerd, minergate, stratum) or
         proc.cmdline contains "stratum+tcp" or
         proc.cmdline contains "pool.minergate")
      output: >
        Crypto mining detected
        (user=%user.name command=%proc.cmdline container=%container.id
        image=%container.image.repository)
      priority: CRITICAL
      tags: [cryptomining, malware]

    - rule: Outbound Connection to Suspicious Port
      desc: Detect outbound connections to known malicious ports
      condition: >
        outbound and
        fd.sport in (4444, 5555, 6666, 9999, 31337) and
        not container.image.repository in (security-scanner)
      output: >
        Suspicious outbound connection
        (user=%user.name command=%proc.cmdline connection=%fd.name
        container=%container.id)
      priority: WARNING
      tags: [network, suspicious]

    - rule: Kubernetes Secret Access
      desc: Detect access to Kubernetes secrets from unexpected processes
      condition: >
        open_read and
        fd.name startswith /var/run/secrets/kubernetes.io and
        not proc.name in (kubectl, kube-proxy, kubelet)
      output: >
        Kubernetes secret accessed
        (user=%user.name command=%proc.cmdline file=%fd.name
        container=%container.id k8s.pod=%k8s.pod.name)
      priority: WARNING
      tags: [kubernetes, secrets]
```

### 5.2 Falco Sidekick for Alert Routing

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: falco-sidekick
  namespace: security
spec:
  replicas: 2
  selector:
    matchLabels:
      app: falco-sidekick
  template:
    metadata:
      labels:
        app: falco-sidekick
    spec:
      containers:
      - name: falco-sidekick
        image: falcosecurity/falcosidekick:2.28.0
        ports:
        - containerPort: 2801
        env:
        - name: SLACK_WEBHOOKURL
          valueFrom:
            secretKeyRef:
              name: falco-sidekick-secrets
              key: slack-webhook
        - name: SLACK_OUTPUTFORMAT
          value: "all"
        - name: SLACK_MINIMUMPRIORITY
          value: "warning"
        - name: ELASTICSEARCH_HOSTPORT
          value: "https://security-siem-es-http:9200"
        - name: ELASTICSEARCH_INDEX
          value: "falco"
        - name: ELASTICSEARCH_MINIMUMPRIORITY
          value: "notice"
        - name: PAGERDUTY_ROUTINGKEY
          valueFrom:
            secretKeyRef:
              name: falco-sidekick-secrets
              key: pagerduty-key
        - name: PAGERDUTY_MINIMUMPRIORITY
          value: "critical"
        - name: PROMETHEUS_ENABLED
          value: "true"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

## 6. Network Security Monitoring

### 6.1 Network Policy Monitoring

```go
package netmon

import (
    "context"
    "encoding/json"
    "time"

    networkingv1 "k8s.io/api/networking/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
)

type NetworkPolicyAuditor struct {
    client     kubernetes.Interface
    auditLog   AuditLogger
}

type NetworkPolicyEvent struct {
    EventType   string                        `json:"event_type"`
    Timestamp   time.Time                     `json:"timestamp"`
    Policy      *networkingv1.NetworkPolicy   `json:"policy"`
    Namespace   string                        `json:"namespace"`
    Actor       string                        `json:"actor"`
    Changes     []PolicyChange                `json:"changes,omitempty"`
}

type PolicyChange struct {
    Field    string `json:"field"`
    OldValue string `json:"old_value"`
    NewValue string `json:"new_value"`
}

func (a *NetworkPolicyAuditor) AuditPolicies(ctx context.Context) error {
    // List all network policies
    policies, err := a.client.NetworkingV1().
        NetworkPolicies("").
        List(ctx, metav1.ListOptions{})
    if err != nil {
        return err
    }

    for _, policy := range policies.Items {
        // Check for overly permissive policies
        findings := a.analyzePolicyRisks(&policy)
        if len(findings) > 0 {
            a.auditLog.Log(ctx, AuditEvent{
                EventType: "network_policy_audit",
                EventName: "risky_policy_detected",
                Severity:  "warning",
                Target: Target{
                    ResourceType: "NetworkPolicy",
                    ResourceID:   string(policy.UID),
                    ResourceName: policy.Name,
                },
                Metadata: map[string]interface{}{
                    "namespace": policy.Namespace,
                    "findings":  findings,
                },
            })
        }
    }

    return nil
}

func (a *NetworkPolicyAuditor) analyzePolicyRisks(
    policy *networkingv1.NetworkPolicy,
) []string {
    var findings []string

    // Check for allow-all ingress
    for _, ingress := range policy.Spec.Ingress {
        if len(ingress.From) == 0 {
            findings = append(findings, "allow_all_ingress")
        }
        for _, from := range ingress.From {
            if from.IPBlock != nil && from.IPBlock.CIDR == "0.0.0.0/0" {
                findings = append(findings, "allow_internet_ingress")
            }
        }
    }

    // Check for allow-all egress
    for _, egress := range policy.Spec.Egress {
        if len(egress.To) == 0 {
            findings = append(findings, "allow_all_egress")
        }
    }

    // Check if policy has no selectors (applies to all pods)
    if len(policy.Spec.PodSelector.MatchLabels) == 0 &&
        len(policy.Spec.PodSelector.MatchExpressions) == 0 {
        findings = append(findings, "applies_to_all_pods")
    }

    return findings
}

// VPC Flow Log Analysis
type FlowLogAnalyzer struct {
    storage    FlowLogStorage
    detector   AnomalyDetector
    alerter    Alerter
}

type FlowLog struct {
    Timestamp       time.Time `json:"timestamp"`
    SourceIP        string    `json:"src_ip"`
    DestinationIP   string    `json:"dst_ip"`
    SourcePort      int       `json:"src_port"`
    DestinationPort int       `json:"dst_port"`
    Protocol        string    `json:"protocol"`
    Bytes           int64     `json:"bytes"`
    Packets         int64     `json:"packets"`
    Action          string    `json:"action"` // ACCEPT, REJECT
    Direction       string    `json:"direction"`
}

func (a *FlowLogAnalyzer) AnalyzeFlows(ctx context.Context, flows []FlowLog) error {
    // Detect port scanning
    portScanners := a.detectPortScanning(flows)
    for ip, scanDetails := range portScanners {
        a.alerter.Alert(ctx, Alert{
            Severity: "high",
            Title:    "Port Scanning Detected",
            Message:  fmt.Sprintf("IP %s scanned %d unique ports", ip, scanDetails.UniquePortCount),
            Metadata: scanDetails,
        })
    }

    // Detect data exfiltration
    exfilCandidates := a.detectDataExfiltration(flows)
    for _, candidate := range exfilCandidates {
        a.alerter.Alert(ctx, Alert{
            Severity: "critical",
            Title:    "Potential Data Exfiltration",
            Message:  fmt.Sprintf("Unusual outbound data volume: %d MB to %s",
                candidate.BytesMB, candidate.DestinationIP),
            Metadata: candidate,
        })
    }

    // Detect lateral movement
    lateralMovement := a.detectLateralMovement(flows)
    if len(lateralMovement) > 0 {
        a.alerter.Alert(ctx, Alert{
            Severity: "high",
            Title:    "Potential Lateral Movement",
            Message:  "Unusual internal network traversal detected",
            Metadata: lateralMovement,
        })
    }

    return nil
}

func (a *FlowLogAnalyzer) detectPortScanning(flows []FlowLog) map[string]*ScanDetails {
    scanners := make(map[string]*ScanDetails)
    portsBySource := make(map[string]map[int]bool)

    for _, flow := range flows {
        if flow.Action == "REJECT" {
            if _, exists := portsBySource[flow.SourceIP]; !exists {
                portsBySource[flow.SourceIP] = make(map[int]bool)
            }
            portsBySource[flow.SourceIP][flow.DestinationPort] = true
        }
    }

    // Flag sources hitting more than 20 unique ports
    for ip, ports := range portsBySource {
        if len(ports) > 20 {
            scanners[ip] = &ScanDetails{
                SourceIP:        ip,
                UniquePortCount: len(ports),
                Ports:           mapKeys(ports),
            }
        }
    }

    return scanners
}
```

## 7. Threat Detection and Response

### 7.1 Threat Detection Pipeline

```python
from dataclasses import dataclass
from typing import List, Dict, Optional
from enum import Enum
import asyncio

class ThreatSeverity(Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

@dataclass
class ThreatIndicator:
    indicator_type: str  # ip, domain, hash, email, url
    value: str
    source: str
    confidence: float
    tags: List[str]
    first_seen: str
    last_seen: str

@dataclass
class DetectedThreat:
    threat_id: str
    timestamp: str
    severity: ThreatSeverity
    threat_type: str
    description: str
    indicators: List[ThreatIndicator]
    affected_assets: List[str]
    evidence: List[Dict]
    recommended_actions: List[str]
    mitre_attack_techniques: List[str]

class ThreatDetectionEngine:
    def __init__(
        self,
        rule_engine,
        ml_detector,
        threat_intel_client,
        correlation_engine
    ):
        self.rule_engine = rule_engine
        self.ml_detector = ml_detector
        self.threat_intel = threat_intel_client
        self.correlation_engine = correlation_engine

    async def analyze_events(
        self,
        events: List[Dict]
    ) -> List[DetectedThreat]:
        """Analyze events through multiple detection methods"""

        threats = []

        # Run detection methods in parallel
        results = await asyncio.gather(
            self._rule_based_detection(events),
            self._ml_based_detection(events),
            self._threat_intel_matching(events),
            self._behavioral_analysis(events)
        )

        # Merge and deduplicate results
        for result_set in results:
            threats.extend(result_set)

        # Correlate threats
        correlated = await self.correlation_engine.correlate(threats)

        return correlated

    async def _rule_based_detection(
        self,
        events: List[Dict]
    ) -> List[DetectedThreat]:
        """Apply SIGMA/YARA rules"""

        threats = []

        for event in events:
            matches = self.rule_engine.match(event)
            for match in matches:
                threats.append(DetectedThreat(
                    threat_id=generate_threat_id(),
                    timestamp=event['timestamp'],
                    severity=ThreatSeverity(match.severity),
                    threat_type=match.threat_type,
                    description=match.description,
                    indicators=[],
                    affected_assets=[event.get('source', {}).get('service')],
                    evidence=[event],
                    recommended_actions=match.recommended_actions,
                    mitre_attack_techniques=match.mitre_techniques
                ))

        return threats

    async def _ml_based_detection(
        self,
        events: List[Dict]
    ) -> List[DetectedThreat]:
        """ML-based anomaly detection"""

        threats = []

        # Extract features for ML model
        features = self._extract_features(events)

        # Run anomaly detection
        anomalies = self.ml_detector.detect(features)

        for anomaly in anomalies:
            if anomaly.score > 0.8:  # High confidence threshold
                original_event = events[anomaly.event_index]
                threats.append(DetectedThreat(
                    threat_id=generate_threat_id(),
                    timestamp=original_event['timestamp'],
                    severity=self._score_to_severity(anomaly.score),
                    threat_type="anomaly",
                    description=f"Anomalous behavior detected: {anomaly.description}",
                    indicators=[],
                    affected_assets=[original_event.get('source', {}).get('service')],
                    evidence=[original_event],
                    recommended_actions=["Investigate anomalous activity"],
                    mitre_attack_techniques=[]
                ))

        return threats

    async def _threat_intel_matching(
        self,
        events: List[Dict]
    ) -> List[DetectedThreat]:
        """Match against threat intelligence feeds"""

        threats = []

        # Extract IOCs from events
        iocs = self._extract_iocs(events)

        # Check against threat intel
        matches = await self.threat_intel.check_batch(iocs)

        for match in matches:
            threats.append(DetectedThreat(
                threat_id=generate_threat_id(),
                timestamp=match.event_timestamp,
                severity=ThreatSeverity.HIGH,
                threat_type="known_threat",
                description=f"Known threat indicator matched: {match.indicator_value}",
                indicators=[ThreatIndicator(
                    indicator_type=match.indicator_type,
                    value=match.indicator_value,
                    source=match.intel_source,
                    confidence=match.confidence,
                    tags=match.tags,
                    first_seen=match.first_seen,
                    last_seen=match.last_seen
                )],
                affected_assets=match.affected_assets,
                evidence=match.evidence,
                recommended_actions=[
                    "Block indicator at perimeter",
                    "Investigate affected systems",
                    "Check for lateral movement"
                ],
                mitre_attack_techniques=match.mitre_techniques
            ))

        return threats

    async def _behavioral_analysis(
        self,
        events: List[Dict]
    ) -> List[DetectedThreat]:
        """Detect threats based on behavioral patterns"""

        threats = []

        # Group events by user/entity
        entities = self._group_by_entity(events)

        for entity_id, entity_events in entities.items():
            # Check for attack patterns
            patterns = self._detect_attack_patterns(entity_events)

            for pattern in patterns:
                threats.append(DetectedThreat(
                    threat_id=generate_threat_id(),
                    timestamp=entity_events[-1]['timestamp'],
                    severity=ThreatSeverity(pattern.severity),
                    threat_type=pattern.attack_type,
                    description=pattern.description,
                    indicators=[],
                    affected_assets=[entity_id],
                    evidence=entity_events,
                    recommended_actions=pattern.recommended_actions,
                    mitre_attack_techniques=pattern.mitre_techniques
                ))

        return threats

    def _detect_attack_patterns(
        self,
        events: List[Dict]
    ) -> List:
        """Detect known attack patterns"""

        patterns = []

        # Pattern: Credential stuffing
        login_failures = [e for e in events
                        if e.get('event_name') == 'login_failure']
        if len(login_failures) > 10:
            unique_usernames = set(e.get('actor', {}).get('username')
                                  for e in login_failures)
            if len(unique_usernames) > 5:
                patterns.append(AttackPattern(
                    attack_type="credential_stuffing",
                    severity="high",
                    description="Multiple failed logins with different usernames",
                    mitre_techniques=["T1110.004"],
                    recommended_actions=[
                        "Enable rate limiting",
                        "Implement CAPTCHA",
                        "Block source IP"
                    ]
                ))

        # Pattern: Privilege escalation chain
        role_changes = [e for e in events
                       if e.get('event_name') == 'role_change']
        sensitive_access = [e for e in events
                          if e.get('event_name') in ['secret_access', 'data_export']]
        if role_changes and sensitive_access:
            patterns.append(AttackPattern(
                attack_type="privilege_escalation_chain",
                severity="critical",
                description="Role change followed by sensitive data access",
                mitre_techniques=["T1078", "T1548"],
                recommended_actions=[
                    "Revoke elevated privileges",
                    "Audit accessed resources",
                    "Reset user credentials"
                ]
            ))

        return patterns
```

### 7.2 Automated Incident Response

```yaml
# SOAR Playbook: Brute Force Response
name: brute_force_response
description: Automated response to brute force attacks
triggers:
  - alert_type: brute_force_authentication
    severity: [high, critical]

steps:
  - name: gather_context
    type: enrichment
    actions:
      - get_user_details:
          user_id: "{{alert.actor.user_id}}"
      - get_ip_reputation:
          ip: "{{alert.actor.ip_address}}"
      - get_recent_events:
          user_id: "{{alert.actor.user_id}}"
          timeframe: 24h

  - name: assess_risk
    type: decision
    conditions:
      - if: "{{ip_reputation.is_malicious}} == true"
        action: block_ip_immediately
      - if: "{{recent_events.failed_logins}} > 20"
        action: lock_account
      - if: "{{user_details.is_privileged}} == true"
        action: escalate_to_security
      - else:
        action: standard_response

  - name: block_ip_immediately
    type: action
    actions:
      - add_to_blocklist:
          ip: "{{alert.actor.ip_address}}"
          duration: 24h
          reason: "Brute force attack - malicious IP"
      - notify:
          channel: security-alerts
          message: "Blocked malicious IP {{alert.actor.ip_address}}"

  - name: lock_account
    type: action
    actions:
      - disable_user:
          user_id: "{{alert.actor.user_id}}"
          reason: "Automated lock - brute force detected"
      - notify_user:
          user_id: "{{alert.actor.user_id}}"
          template: account_locked_security
      - create_ticket:
          type: security_incident
          priority: high

  - name: escalate_to_security
    type: action
    actions:
      - page_oncall:
          team: security
          message: "Privileged account under brute force attack"
      - create_war_room:
          name: "Incident-{{alert.id}}"
          invite: [security, sre]

  - name: standard_response
    type: action
    actions:
      - add_to_watchlist:
          ip: "{{alert.actor.ip_address}}"
          duration: 1h
      - enable_mfa_challenge:
          user_id: "{{alert.actor.user_id}}"
      - notify:
          channel: security-monitoring
          message: "Brute force attempt on {{alert.actor.username}}"

  - name: document_incident
    type: action
    run: always
    actions:
      - create_incident_report:
          alert: "{{alert}}"
          context: "{{gather_context}}"
          actions_taken: "{{steps_executed}}"
      - update_metrics:
          metric: security_incidents_total
          labels:
            type: brute_force
            severity: "{{alert.severity}}"
```

## 8. Security Metrics and Dashboards

### 8.1 Security KPIs

```yaml
# Security Metrics Definition
security_metrics:
  detection:
    - name: mean_time_to_detect
      description: "Average time from threat occurrence to detection"
      target: "< 5 minutes"
      calculation: "avg(detection_time - event_time)"

    - name: detection_coverage
      description: "Percentage of security events captured"
      target: "> 99%"
      calculation: "captured_events / total_events * 100"

    - name: false_positive_rate
      description: "Percentage of alerts that are false positives"
      target: "< 5%"
      calculation: "false_positive_alerts / total_alerts * 100"

    - name: detection_accuracy
      description: "True positive rate for threat detection"
      target: "> 95%"
      calculation: "true_positives / (true_positives + false_negatives)"

  response:
    - name: mean_time_to_respond
      description: "Average time from detection to response initiation"
      target: "< 15 minutes"
      calculation: "avg(response_time - detection_time)"

    - name: mean_time_to_contain
      description: "Average time to contain active threats"
      target: "< 1 hour"
      calculation: "avg(containment_time - detection_time)"

    - name: mean_time_to_remediate
      description: "Average time to fully remediate incidents"
      target: "< 24 hours"
      calculation: "avg(remediation_time - detection_time)"

  coverage:
    - name: asset_monitoring_coverage
      description: "Percentage of assets with security monitoring"
      target: "100%"

    - name: vulnerability_scan_coverage
      description: "Percentage of assets scanned for vulnerabilities"
      target: "> 95%"

    - name: log_collection_coverage
      description: "Percentage of systems sending security logs"
      target: "100%"

  compliance:
    - name: policy_compliance_rate
      description: "Percentage of systems meeting security policy"
      target: "> 98%"

    - name: patch_compliance_rate
      description: "Percentage of systems with current patches"
      target: "> 95%"

    - name: access_review_completion
      description: "Percentage of required access reviews completed"
      target: "100%"
```

### 8.2 Grafana Dashboard

```json
{
  "dashboard": {
    "title": "Security Operations Center",
    "tags": ["security", "soc"],
    "panels": [
      {
        "title": "Security Alert Overview",
        "type": "stat",
        "gridPos": {"x": 0, "y": 0, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "sum(increase(security_alerts_total[24h]))",
            "legendFormat": "Total Alerts (24h)"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "thresholds": {
              "mode": "absolute",
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 50},
                {"color": "red", "value": 100}
              ]
            }
          }
        }
      },
      {
        "title": "Critical Alerts",
        "type": "stat",
        "gridPos": {"x": 6, "y": 0, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "sum(increase(security_alerts_total{severity=\"critical\"}[24h]))",
            "legendFormat": "Critical"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "color": {"mode": "thresholds"},
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "red", "value": 1}
              ]
            }
          }
        }
      },
      {
        "title": "MTTD (Mean Time to Detect)",
        "type": "gauge",
        "gridPos": {"x": 12, "y": 0, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "avg(security_detection_time_seconds)",
            "legendFormat": "MTTD"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "s",
            "max": 600,
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 180},
                {"color": "red", "value": 300}
              ]
            }
          }
        }
      },
      {
        "title": "MTTR (Mean Time to Respond)",
        "type": "gauge",
        "gridPos": {"x": 18, "y": 0, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "avg(security_response_time_seconds)",
            "legendFormat": "MTTR"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "s",
            "max": 1800,
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 600},
                {"color": "red", "value": 900}
              ]
            }
          }
        }
      },
      {
        "title": "Alerts by Severity (24h)",
        "type": "piechart",
        "gridPos": {"x": 0, "y": 4, "w": 8, "h": 8},
        "targets": [
          {
            "expr": "sum by (severity) (increase(security_alerts_total[24h]))",
            "legendFormat": "{{severity}}"
          }
        ]
      },
      {
        "title": "Alerts by Type",
        "type": "timeseries",
        "gridPos": {"x": 8, "y": 4, "w": 16, "h": 8},
        "targets": [
          {
            "expr": "sum by (alert_type) (rate(security_alerts_total[5m]))",
            "legendFormat": "{{alert_type}}"
          }
        ]
      },
      {
        "title": "Active Threats",
        "type": "table",
        "gridPos": {"x": 0, "y": 12, "w": 24, "h": 8},
        "targets": [
          {
            "expr": "security_active_threats",
            "format": "table"
          }
        ],
        "transformations": [
          {
            "type": "organize",
            "options": {
              "columns": ["threat_id", "severity", "threat_type", "affected_asset", "status", "age"]
            }
          }
        ]
      },
      {
        "title": "Failed Logins by Source",
        "type": "geomap",
        "gridPos": {"x": 0, "y": 20, "w": 12, "h": 10},
        "targets": [
          {
            "expr": "sum by (country) (increase(auth_login_failures_total[1h]))",
            "legendFormat": "{{country}}"
          }
        ]
      },
      {
        "title": "Top Blocked IPs",
        "type": "table",
        "gridPos": {"x": 12, "y": 20, "w": 12, "h": 10},
        "targets": [
          {
            "expr": "topk(10, sum by (ip, reason) (firewall_blocked_total))",
            "format": "table"
          }
        ]
      },
      {
        "title": "Falco Alerts",
        "type": "timeseries",
        "gridPos": {"x": 0, "y": 30, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "sum by (rule) (rate(falco_events_total[5m]))",
            "legendFormat": "{{rule}}"
          }
        ]
      },
      {
        "title": "Security Compliance Score",
        "type": "gauge",
        "gridPos": {"x": 12, "y": 30, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "security_compliance_score",
            "legendFormat": "Compliance"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 0,
            "max": 100,
            "thresholds": {
              "steps": [
                {"color": "red", "value": null},
                {"color": "yellow", "value": 80},
                {"color": "green", "value": 95}
              ]
            }
          }
        }
      }
    ]
  }
}
```

## 9. Compliance Monitoring

### 9.1 Compliance Framework Mapping

```yaml
# Compliance Controls Mapping
compliance_frameworks:
  SOC2:
    security_monitoring_controls:
      CC6.1:
        description: "Logical and physical access controls"
        monitoring_requirements:
          - authentication_logging
          - access_control_audit
          - session_monitoring
        evidence:
          - "Authentication logs with user, timestamp, outcome"
          - "Access control change audit trail"
          - "Session duration and activity logs"

      CC6.6:
        description: "Security event monitoring"
        monitoring_requirements:
          - security_event_collection
          - alert_generation
          - incident_tracking
        evidence:
          - "SIEM event collection reports"
          - "Alert configuration and history"
          - "Incident tickets and resolution"

      CC7.2:
        description: "System monitoring"
        monitoring_requirements:
          - performance_monitoring
          - availability_monitoring
          - anomaly_detection
        evidence:
          - "System health dashboards"
          - "Uptime reports"
          - "Anomaly detection logs"

  PCI_DSS:
    security_monitoring_controls:
      "10.1":
        description: "Audit trails for system components"
        monitoring_requirements:
          - user_action_logging
          - admin_action_logging
          - system_event_logging

      "10.2":
        description: "Automated audit trails"
        monitoring_requirements:
          - individual_user_access
          - actions_by_privileged_users
          - access_to_audit_trails
          - invalid_access_attempts
          - authentication_mechanism_use
          - initialization_of_audit_logs
          - creation_deletion_of_system_objects

      "10.6":
        description: "Review logs daily"
        monitoring_requirements:
          - daily_log_review_process
          - automated_log_analysis
          - exception_reporting

  GDPR:
    security_monitoring_controls:
      article_32:
        description: "Security of processing"
        monitoring_requirements:
          - access_logging
          - data_access_monitoring
          - breach_detection

      article_33:
        description: "Breach notification"
        monitoring_requirements:
          - breach_detection_capability
          - notification_workflow
          - documentation_process
```

### 9.2 Compliance Automation

```python
from dataclasses import dataclass
from typing import List, Dict, Optional
from datetime import datetime, timedelta

@dataclass
class ComplianceControl:
    control_id: str
    framework: str
    description: str
    requirements: List[str]
    evidence_sources: List[str]
    validation_query: str

@dataclass
class ComplianceCheckResult:
    control_id: str
    status: str  # compliant, non_compliant, partial, not_applicable
    evidence: List[Dict]
    findings: List[str]
    last_checked: datetime
    next_review: datetime

class ComplianceMonitor:
    def __init__(
        self,
        siem_client,
        audit_client,
        evidence_storage
    ):
        self.siem = siem_client
        self.audit = audit_client
        self.evidence = evidence_storage

    async def check_compliance(
        self,
        framework: str,
        control_ids: Optional[List[str]] = None
    ) -> List[ComplianceCheckResult]:
        """Run compliance checks for specified controls"""

        controls = self._get_controls(framework, control_ids)
        results = []

        for control in controls:
            result = await self._check_control(control)
            results.append(result)

            # Store evidence
            await self.evidence.store(
                control_id=control.control_id,
                check_time=datetime.utcnow(),
                result=result
            )

        return results

    async def _check_control(
        self,
        control: ComplianceControl
    ) -> ComplianceCheckResult:
        """Check individual compliance control"""

        findings = []
        evidence = []

        # Execute validation query
        query_results = await self.siem.query(control.validation_query)
        evidence.append({
            "type": "siem_query",
            "query": control.validation_query,
            "results": query_results
        })

        # Check each requirement
        for requirement in control.requirements:
            check_result = await self._check_requirement(requirement)
            if not check_result.passed:
                findings.append(f"{requirement}: {check_result.reason}")
            evidence.append({
                "type": "requirement_check",
                "requirement": requirement,
                "result": check_result
            })

        # Determine overall status
        if not findings:
            status = "compliant"
        elif len(findings) < len(control.requirements) / 2:
            status = "partial"
        else:
            status = "non_compliant"

        return ComplianceCheckResult(
            control_id=control.control_id,
            status=status,
            evidence=evidence,
            findings=findings,
            last_checked=datetime.utcnow(),
            next_review=datetime.utcnow() + timedelta(days=30)
        )

    async def _check_requirement(self, requirement: str):
        """Check specific requirement"""

        checks = {
            "authentication_logging": self._check_auth_logging,
            "access_control_audit": self._check_access_audit,
            "security_event_collection": self._check_event_collection,
            "daily_log_review_process": self._check_log_review,
            "breach_detection_capability": self._check_breach_detection,
        }

        checker = checks.get(requirement)
        if checker:
            return await checker()

        return CheckResult(passed=True, reason="Not implemented")

    async def _check_auth_logging(self):
        """Verify authentication logging is enabled and working"""

        # Check for recent auth events
        recent_events = await self.siem.query(
            """
            event_type:authentication
            | stats count by event_name
            | where _time > now() - 1h
            """
        )

        if not recent_events:
            return CheckResult(
                passed=False,
                reason="No authentication events in last hour"
            )

        # Verify required fields present
        required_fields = ['user_id', 'timestamp', 'outcome', 'ip_address']
        sample_event = await self.siem.get_sample_event('authentication')

        missing_fields = [f for f in required_fields if f not in sample_event]
        if missing_fields:
            return CheckResult(
                passed=False,
                reason=f"Missing required fields: {missing_fields}"
            )

        return CheckResult(passed=True, reason="All checks passed")

    async def generate_compliance_report(
        self,
        framework: str,
        period_start: datetime,
        period_end: datetime
    ) -> Dict:
        """Generate compliance report for auditors"""

        results = await self.check_compliance(framework)

        # Calculate overall compliance score
        compliant_count = sum(1 for r in results if r.status == "compliant")
        total_count = len(results)
        compliance_score = (compliant_count / total_count) * 100

        # Get historical evidence
        evidence_records = await self.evidence.get_period(
            framework=framework,
            start=period_start,
            end=period_end
        )

        return {
            "framework": framework,
            "period": {
                "start": period_start.isoformat(),
                "end": period_end.isoformat()
            },
            "overall_score": compliance_score,
            "summary": {
                "compliant": compliant_count,
                "non_compliant": sum(1 for r in results if r.status == "non_compliant"),
                "partial": sum(1 for r in results if r.status == "partial"),
                "total": total_count
            },
            "control_results": [
                {
                    "control_id": r.control_id,
                    "status": r.status,
                    "findings": r.findings,
                    "last_checked": r.last_checked.isoformat()
                }
                for r in results
            ],
            "evidence_summary": {
                "total_records": len(evidence_records),
                "storage_location": self.evidence.get_location(framework)
            },
            "generated_at": datetime.utcnow().isoformat()
        }
```

## 10. Implementation Checklist

### 10.1 Phase 1: Foundation (Weeks 1-4)

- [ ] Deploy security logging infrastructure
  - [ ] Configure structured security log format
  - [ ] Implement audit trail with integrity verification
  - [ ] Set up log shipping to SIEM
- [ ] Implement basic threat detection
  - [ ] Deploy Falco for runtime security
  - [ ] Configure authentication monitoring
  - [ ] Set up brute force detection rules
- [ ] Create security dashboards
  - [ ] Alert overview dashboard
  - [ ] Authentication monitoring dashboard

### 10.2 Phase 2: Detection (Weeks 5-8)

- [ ] Enhance threat detection capabilities
  - [ ] Implement ML-based anomaly detection
  - [ ] Integrate threat intelligence feeds
  - [ ] Configure behavioral analysis
- [ ] Deploy network security monitoring
  - [ ] VPC flow log analysis
  - [ ] Network policy auditing
- [ ] Implement automated enrichment
  - [ ] GeoIP enrichment
  - [ ] User context enrichment
  - [ ] Risk scoring

### 10.3 Phase 3: Response (Weeks 9-12)

- [ ] Implement SOAR capabilities
  - [ ] Automated response playbooks
  - [ ] Incident ticketing integration
  - [ ] Communication automation
- [ ] Deploy compliance monitoring
  - [ ] Control mapping configuration
  - [ ] Automated compliance checks
  - [ ] Evidence collection automation
- [ ] Create security operations procedures
  - [ ] Incident response runbooks
  - [ ] Escalation procedures
  - [ ] Post-incident review process

## 11. Related Specifications

| Specification | Relationship |
|--------------|--------------|
| [SRG-002 Logging](SRG-002%20Logging.md) | Foundation for security event logging |
| [SRG-003 Error Tracking](SRG-003%20Error%20Tracking.md) | Integration with error monitoring |
| [SRG-005 Alerting Rules](SRG-005%20Alerting%20Rules.md) | Security alert routing and escalation |
| [SRO-004 Multi-Region DR](SRO-004%20Multi-Region%20DR.md) | DR for security infrastructure |
| [SRO-002 Chaos Engineering](SRO-002%20Chaos%20Engineering.md) | Security chaos testing |

---

**Document History:**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2024-01-15 | Security Team | Initial specification |
