# SRG-005 Alerting Rules (Правила алертинга)

## Определение

Alerting Rules - это условия, при которых система мониторинга генерирует оповещения о проблемах. Хороший алерт - это четкое, своевременное и действуемое уведомление.

## Типы алертов

### 1. Критичные алерты (Critical/P1)

Требуют немедленного действия (в течение 5-15 минут).

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

### 2. Предупреждающие алерты (Warning/P2)

Требуют действия в течение 1-4 часов.

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

### 3. Информационные алерты (Info/P3)

Не требуют немедленного действия, но нужно отслеживать тренды.

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

## Правила хороших алертов

### 1. Алерт должен быть действуемым

❌ Не делать:
```yaml
- alert: CPUUsage
  expr: cpu_usage > 0
  # CPU всегда > 0, это нормально!
```

✅ Делать:
```yaml
- alert: HighCPUUsage
  expr: cpu_usage > 80
  for: 10m
  # Это проблема, которую можно решить
```

### 2. Настроить время предварительной проверки (for)

❌ Не делать:
```yaml
- alert: ServiceDown
  for: 0m  # Сработает при каждом перезапуске
  expr: up == 0
```

✅ Делать:
```yaml
- alert: ServiceDown
  for: 5m  # Сработает только если сервис не поднимается
  expr: up == 0
```

### 3. Использовать rate вместо raw counters

❌ Не делать:
```yaml
- alert: TooManyErrors
  expr: http_requests_total{status="500"} > 1000
  # counter всегда растет!
```

✅ Делать:
```yaml
- alert: HighErrorRate
  expr: rate(http_requests_total{status="500"}[5m]) > 10
  # errors per second за последние 5 минут
```

### 4. Использовать quantiles для latency

❌ Не делать:
```yaml
- alert: HighLatency
  expr: http_request_duration_seconds > 0.5
  # Averages can be misleading
```

✅ Делать:
```yaml
- alert: HighLatency
  expr: |
    histogram_quantile(0.95,
      rate(http_request_duration_seconds_bucket[5m])
    ) > 0.5
  # 95% of requests are under 500ms
```

### 5. Алерт должен быть конкретным

❌ Не делать:
```yaml
- alert: SomethingWrong
  expr: (metric1 > 10) OR (metric2 < 5) OR (metric3 == 0)
  # Слишком сложно, непонятно что делать
```

✅ Делать:
```yaml
- alert: ServiceDown
  expr: up == 0
  # Четко, понятно что делать
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
  team: platform          # Кто отвечает
  service: order-service  # Какой сервис
  environment: production # Где сработало
  component: api         # Компонент
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

## Мета-алерты

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

## Конфигурация через переменные окружения

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

✅ **Делать**
* Тестировать алерты при нагрузке
* Иметь runbook для каждого алерта
* Использовать разные каналы для разной важности
* Анализировать false positive алерты еженедельно
* Документировать причины каждого алерта
* Использовать `for` для избежания ложных срабатываний
* Иметь метрику alert: время от проблемы до алерта
* Review и refine алерты каждый квартал

❌ **Не делать**
* Алерт на каждую метрику
* Отправлять все алерты в Slack
* Являться единственным получателем алертов
* Игнорировать алерты (если игнорируете, удалите)
* Паниковать по каждому алерту
* Использовать алерты для информации (используйте дашборды)
* Оставлять silences навсегда (fix the problem)

## Дополнительные ресурсы

* [Prometheus Alerting](https://prometheus.io/docs/alerting/latest/overview/)
* [Alertmanager Documentation](https://prometheus.io/docs/alerting/latest/alertmanager/)
* [My Philosophy on Alerting](https://docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8e2S8m5baEZNw1EAWxw/edit)
* [Site Reliability Engineering Book, Chapter Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/)
* [The Practical Guide to Anomaly Detection](https://www.pagerduty.com/resources/learn/anomaly-detection-guide/)
