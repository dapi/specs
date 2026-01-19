# SRG-005 Alerting Rules

## Definition

Alerting Rules are conditions under which a monitoring system generates alerts about issues. A good alert is a clear, timely, and actionable notification.

## Alert types

### 1. Critical alerts (Critical/P1)

Require immediate action (within 5-15 minutes).

```yaml
# Service down
- alert: ServiceDown
  for: 5m
  expr: up{job="order-service"} == 0
  labels:
    severity: critical
    team: platform
  annotations:
    summary: "Order service is down"
    description: "Order service has been down for more than 5 minutes"
    runbook: "https://wiki.example.com/runbooks/service-down"

# High error rate
- alert: HighErrorRate
  for: 5m
  expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
  labels:
    severity: critical
    team: platform
  annotations:
    summary: "High error rate detected"
    description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.service }}"
```

**Action**: Page on-call engineer, investigate immediately.

### 2. Warning alerts (Warning/P2)

Require action within 1-4 hours.

```yaml
# High latency
- alert: HighLatency
  for: 10m
  expr: |
    histogram_quantile(0.95,
      rate(http_request_duration_seconds_bucket[5m])
    ) > 0.5
  labels:
    severity: warning
    team: platform
  annotations:
    summary: "High latency detected"
    description: "95th percentile latency is {{ $value }}s for {{ $labels.service }}"

# High resource usage
- alert: HighCPUUsage
  for: 15m
  expr: 100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
  labels:
    severity: warning
    team: infra
  annotations:
    summary: "High CPU usage"
    description: "CPU usage is {{ $value }}% on {{ $labels.instance }}"
```

**Action**: Investigate during business hours.

### 3. Informational alerts (Info/P3)

Do not require immediate action, but need to track trends.

```yaml
# Certificate expiration
- alert: CertificateExpiringSoon
  for: 1h
  expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
  labels:
    severity: info
    team: platform
  annotations:
    summary: "Certificate expiring soon"
    description: "Certificate for {{ $labels.instance }} expires in {{ $value }} days"

# Deployment
- alert: DeploymentInProgress
  for: 0m  # Trigger immediately
  expr: changes(deployment_status{condition="available"}[5m]) > 0
  labels:
    severity: info
    team: platform
  annotations:
    summary: "Deployment started"
    description: "New deployment for {{ $labels.deployment }}"
```

**Action**: No immediate action, track trends.

## Good alerting rules

### 1. Alert must be actionable

❌ Don't:
```yaml
- alert: CPUUsage
  expr: cpu_usage > 0
  # CPU is always > 0, this is normal!
```

✅ Do:
```yaml
- alert: HighCPUUsage
  expr: cpu_usage > 80
  for: 10m
  # This is a problem that can be solved
```

### 2. Configure evaluation period (for)

❌ Don't:
```yaml
- alert: ServiceDown
  for: 0m  # Will fire on every restart
  expr: up == 0
```

✅ Do:
```yaml
- alert: ServiceDown
  for: 5m  # Will only fire if service doesn't recover
  expr: up == 0
```

### 3. Use rate instead of raw counters

❌ Don't:
```yaml
- alert: TooManyErrors
  expr: http_requests_total{status="500"} > 1000
  # counter always increases!
```

✅ Do:
```yaml
- alert: HighErrorRate
  expr: rate(http_requests_total{status="500"}[5m]) > 10
  # errors per second for the last 5 minutes
```

### 4. Use quantiles for latency

❌ Don't:
```yaml
- alert: HighLatency
  expr: http_request_duration_seconds > 0.5
  # Averages can be misleading
```

✅ Do:
```yaml
- alert: HighLatency
  expr: |
    histogram_quantile(0.95,
      rate(http_request_duration_seconds_bucket[5m])
    ) > 0.5
  # 95% of requests are under 500ms
```

### 5. Alert must be specific

❌ Don't:
```yaml
- alert: SomethingWrong
  expr: (metric1 > 10) OR (metric2 < 5) OR (metric3 == 0)
  # Too complex, unclear what to do
```

✅ Do:
```yaml
- alert: ServiceDown
  expr: up == 0
  # Clear, obvious what to do
```

## Avoiding Alert Fatigue

### 1. Group related alerts

```yaml
groups:
- name: service.alerts
  interval: 30s
  rules:
  # API alerts
  - alert: APIHighErrorRate
    expr: ...

  - alert: APIHighLatency
    expr: ...

- name: infrastructure.alerts
  interval: 1m
  rules:
  # Node alerts
  - alert: NodeHighCPU
    expr: ...
```

### 2. Use inhibition rules

```yaml
# Alertmanager config
inhibit_rules:
- source_match:
    severity: 'critical'
  target_match:
    severity: 'warning'
  equal: ['alertname', 'cluster', 'service']
  # If critical alert is firing, mute warnings for same service
```

### 3. Set appropriate intervals

```yaml
# Check every minute (default)
groups:
- name: alerts
  interval: 1m

# Check every 5 seconds (critical)
- name: critical
  interval: 5s

# Check every 10 minutes (non-critical)
- name: informational
  interval: 10m
```

## Alert structure

### Labels

```yaml
labels:
  severity: critical      # critical|warning|info
  team: platform          # Who is responsible
  service: order-service  # Which service
  environment: production # Where it fired
  component: api         # Component
```

### Annotations

```yaml
annotations:
  # Short description
  summary: "Order service is down"

  # Detailed description (can use templates)
  description: |
    Order service {{ $labels.service }} on {{ $labels.environment }}
    has been down for {{ $value }} minutes

  # How to fix
  runbook: "https://wiki.example.com/runbooks/service-down"

  # Additional context
  dashboard: "https://grafana.example.com/d/service/{{ $labels.service }}"
  logs: "https://kibana.example.com/app/logs?service={{ $labels.service }}"
```

## Alert routing

### PagerDuty

```yaml
receivers:
- name: 'pagerduty-critical'
  pagerduty_configs:
  - routing_key: '<integration-key>'
    severity: '{{ .GroupLabels.severity }}'
    custom_details:
      service: '{{ .GroupLabels.service }}'
      environment: '{{ .GroupLabels.environment }}'
      description: '{{ .CommonAnnotations.description }}'
      runbook: '{{ .CommonAnnotations.runbook }}'
```

### Slack

```yaml
receivers:
- name: 'slack-warnings'
  slack_configs:
  - api_url: '<slack-webhook>'
    channel: '#alerts-warning'
    title: '[{{ .GroupLabels.severity | upper }}] {{ .GroupLabels.alertname }}'
    text: |
      {{ range .Alerts }}
      *Service:* {{ .Labels.service }}
      *Environment:* {{ .Labels.environment }}
      *Description:* {{ .Annotations.description }}
      *Runbook:* {{ .Annotations.runbook }}
      {{ end }}
    color: '{{ if eq .GroupLabels.severity "critical" }}danger{{ else }}warning{{ end }}'
```

### Email

```yaml
receivers:
- name: 'email-info'
  email_configs:
  - to: 'team@example.com'
    subject: '[{{ .GroupLabels.severity }}] {{ .GroupLabels.alertname }}'
    body: |
      {{ range .Alerts }}
      Alert: {{ .Annotations.summary }}
      Description: {{ .Annotations.description }}
      Labels: {{ range .Labels.SortedPairs }}{{ .Name }}={{ .Value }} {{ end }}
      {{ end }}
```

### Routing logic

```yaml
route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'default'

  routes:
  # Critical alerts -> PagerDuty
  - match:
      severity: critical
    receiver: 'pagerduty-critical'
    continue: false

  # Warnings -> Slack
  - match:
      severity: warning
    receiver: 'slack-warnings'
    continue: false

  # Info -> Email (business hours only)
  - match:
      severity: info
    receiver: 'email-info'
    mute_time_intervals:
    - non_business_hours

mute_time_intervals:
- name: non_business_hours
  time_intervals:
  - times:
    - start_time: '18:00'
      end_time: '09:00'
    weekdays: ['saturday', 'sunday']
```

## Meta-alerts

### Disabled alerts

```yaml
- alert: AlertmanagerClusterDown
  expr: up{job="alertmanager"} == 0
  for: 5m
  labels:
    severity: critical
```

### Silent alerts

```yaml
- alert: AlertmanagerSilencesActive
  expr: alertmanager_silences{state="active"} > 0
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "{{ $value }} active silences"
```

## Testing alerts

### Unit tests

```yaml
# alert_tests.yaml
group_eval_order:
  - service.alerts

tests:
- interval: 1m
  input_series:
  - series: 'up{job="order-service"}'
    values: '1 1 0 0 0 0 0'

  alert_rule_test:
  - eval_time: 5m
    alertname: ServiceDown
    exp_alerts:
    - exp_labels:
        job: order-service
        severity: critical
      exp_annotations:
        summary: "Order service is down"
```

Run tests:
```bash
promtool test rules alert_tests.yaml
```

### Integration tests

```bash
# Trigger alert condition
curl -X POST http://service/api/trigger-error

# Check alert fired
curl http://alertmanager:9093/api/v1/alerts

# Verify notification received
curl http://mock-slack:8080/messages
```

## Configuration via environment variables

```
# Alerting
ALERTING_ENABLED=true
ALERTING_MANAGER_URL=http://alertmanager:9093

# Routes
ALERTING_ROUTE_GROUP_BY=alertname,cluster,service
ALERTING_ROUTE_GROUP_WAIT=10s
ALERTING_ROUTE_GROUP_INTERVAL=10s
ALERTING_ROUTE_REPEAT_INTERVAL=1h

# PagerDuty
ALERTING_PAGERDUTY_ENABLED=true
ALERTING_PAGERDUTY_ROUTING_KEY=<key>
ALERTING_PAGERDUTY_SEVERITY_TEMPLATE='{{ .GroupLabels.severity }}'

# Slack
ALERTING_SLACK_ENABLED=true
ALERTING_SLACK_WEBHOOK_URL=https://hooks.slack.com/...
ALERTING_SLACK_CHANNEL='#alerts'

# Email
ALERTING_EMAIL_ENABLED=true
ALERTING_EMAIL_TO=team@example.com
ALERTING_EMAIL_FROM=alerts@example.com
```

## Best practices

✅ **Do**
* Test alerts under load
* Have runbook for each alert
* Use different channels for different severities
* Analyze false positive alerts weekly
* Document reasons for each alert
* Use `for` to avoid false positives
* Have a metric for alert: time from problem to alert
* Review and refine alerts every quarter

❌ **Don't**
* Alert on every metric
* Send all alerts to Slack
* Be the only recipient of alerts
* Ignore alerts (if you ignore them, remove them)
* Panic over every alert
* Use alerts for information (use dashboards instead)
* Leave silences forever (fix the problem)

## Additional resources

* [Prometheus Alerting](https://prometheus.io/docs/alerting/latest/overview/)
* [Alertmanager Documentation](https://prometheus.io/docs/alerting/latest/alertmanager/)
* [My Philosophy on Alerting](https://docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8e2S8m5baEZNw1EAWxw/edit)
* [Site Reliability Engineering Book, Chapter Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/)
* [The Practical Guide to Anomaly Detection](https://www.pagerduty.com/resources/learn/anomaly-detection-guide/)
