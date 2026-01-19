# SRG-007 SLI/SLO/SLA (Уровни обслуживания)

SLI/SLO/SLA - это фреймворк для определения, измерения и управления надежностью и производительностью сервисов.

- **SLI** (Service Level Indicator) - метрика, измеряющая уровень обслуживания
- **SLO** (Service Level Objective) - целевое значение SLI
- **SLA** (Service Level Agreement) - юридически обязывающее соглашение с клиентами

---

## Основные концепции

### Service Level Indicator (SLI)

**Определение:** Количественная метрика, измеряющая уровень обслуживания.

**Формула:**
```
SLI = (Хорошие события / Всего событий) × 100%
```

**Примеры SLI:**

| Категория | SLI | Формула |
|-----------|-----|---------|
| Доступность | Uptime % | `(Успешные запросы / Всего запросов) × 100` |
| Производительность | Latency | `P99 < 200ms` |
| Надежность | Error rate | `(5xx ошибки / Всего запросов) × 100` |
| Пропускная способность | Throughput | `Запросов в секунду` |
| Данные | Freshness | `Возраст данных < 5 минут` |
| Корректность | Correctness | `(Правильные ответы / Всего) × 100` |

**Практические SLI:**

```python
# Пример вычисления SLI для HTTP API
class SLICalculator:
    def __init__(self, lookback_window='28d'):
        self.lookback_window = lookback_window

    def calculate_availability(self):
        """Доступность сервиса"""
        total_requests = metrics.get('http_requests_total')
        failed_requests = metrics.get('http_requests_5xx')

        return ((total_requests - failed_requests) / total_requests) * 100

    def calculate_latency_sli(self):
        """Производительность по latency"""
        p99_latency = metrics.get('http_request_duration_p99')
        sla_target = 200  # ms

        return (sla_target / p99_latency) * 100 if p99_latency > 0 else 100

    def calculate_good_requests(self):
        """"Хорошие" запросы по всем параметрам"""
        return metrics.query("""
            http_requests_total
            - http_requests_5xx
            - http_requests_4xx
            - http_requests_duration > 200ms
        """)
```

---

### Service Level Objective (SLO)

**Определение:** Целевое значение SLI за определенный период времени.

**Формат:**
```
SLO = (Хорошие события / Всего событий) ≥ Target%
```

**Типичные SLO:**

| Сервис | SLO | Период | Допустимый простой |
|--------|-----|--------|-------------------|
| Пользовательский API | 99.9% uptime | 28 дней | 43 минуты в месяц |
| Внутренний API | 99.5% uptime | 28 дней | 3.5 часа в месяц |
| Платежный шлюз | 99.99% uptime | 28 дней | 4.3 минуты в месяц |
| Фронтенд | P99 < 200ms | 7 дней | 1% запросов медленнее |

**Пример SLO:**

```yaml
# slo-payment-service.yaml
apiVersion: monitoring.googleapis.com/v1
kind: ServiceLevelObjective
metadata:
  name: payments-availability
spec:
  service: payment-service
  description: "Платежный шлюз должен быть доступен 99.99% времени"
  sli:
    availability:
      requestBased:
        goodTotalRatio:
          goodServiceFilter: 'metric.type="custom.googleapis.com/http/2xx"'
          totalServiceFilter: 'metric.type="custom.googleapis.com/http/all"'
  goal: 0.9999  # 99.99%
  calendarPeriod: 28d
  alerting_rules:
    - burn_rate_threshold: 14.4
      window: 1h
    - burn_rate_threshold: 6
      window: 6h
```

**Error Budget:**
```
Error Budget = 100% - SLO

Для 99.9% SLO:
Error Budget = 0.1%
В месяц: 43.8 минут простоя / 10.1 часа в год
```

---

### Service Level Agreement (SLA)

**Определение:** Юридически обязывающее соглашение с клиентами о компенсациях за невыполнение SLO.

**Пример SLA:**

```markdown
## Доступность API

**SLO:** 99.9% uptime в календарный месяц

**SLA:**

| Уровень простоя | Кредит сервиса | Компенсация |
|-----------------|----------------|-------------|
| < 0.1%          | 0%             | None        |
| 0.1% - 0.5%     | 10%            | $100 credit |
| 0.5% - 1%       | 25%            | $250 credit |
| > 1%            | 50%            | $500 credit |

**Исключения:**
- Planned maintenance (объявлена за 7 дней)
- Force majeure
- Third-party provider outages
- Customer misuse
```

---

## Процесс определения SLI/SLO/SLA

### Шаг 1: Определить критичность сервиса

```yaml
service_classification:
  tier_1_critical:
    examples: ["payment_gateway", "auth_service"]
    slos: ["99.99% uptime", "P99 latency < 200ms"]
    on_call: "24/7 immediate"

  tier_2_important:
    examples: ["recommendations", "search"]
    slos: ["99.9% uptime", "P95 latency < 500ms"]
    on_call: "business hours < 1h"

  tier_3_best_effort:
    examples: ["analytics", "logs"]
    slos: ["99% uptime"]
    on_call: "business hours next day"
```

### Шаг 2: Выбрать показатели (SLIs)

**For API Services:**
```yaml
slis:
  availability:
    description: "Доля успешных запросов"
    query: 'rate(http_requests_success[5m]) / rate(http_requests_total[5m])'

  latency:
    description: "Latency процентили"
    queries:
      p50: 'histogram_quantile(0.5, rate(http_request_duration_seconds_bucket[5m]))'
      p95: 'histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))'
      p99: 'histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))'

  error_rate:
    description: "Доля ошибочных запросов"
    query: 'rate(http_requests_5xx[5m]) / rate(http_requests_total[5m])'
```

**For Data Pipeline:**
```yaml
slis:
  freshness:
    description: "Возраст данных"
    query: 'time() - max(data_timestamp)'

  completeness:
    description: "Полнота данных"
    query: 'count_valid_records / count_total_records'

  correctness:
    description: "Корректность данных"
    query: 'count_correct_records / count_total_records'
```

**For Queue/Job Systems:**
```yaml
slis:
  throughput:
    description: "Сообщений в секунду"
    query: 'rate(queue_messages_processed[5m])'

  age:
    description: "Возраст самого старого сообщения"
    query: 'max(queue_message_age_seconds)'

  error_rate:
    description: "Доля ошибочных выполнений"
    query: 'rate(queue_jobs_failed[5m]) / rate(queue_jobs_total[5m])'
```

### Шаг 3: Определить цели (SLOs)

**90th percentile target:**
```yaml
# Для большинства запросов
slo_90pct:
  slis: [availability, latency]
  targets:
    availability: 99.9%  # 0.1% ошибок
    latency_p99: 200ms
    latency_p95: 100ms
    latency_p50: 50ms

  window: 28d
  error_budget: 0.1%  # 43.8 минут в месяц
```

**Типичные SLO по индустрии:**

| Сервис | Доступность | Latency P99 | Error Rate |
|--------|-------------|-------------|------------|
| Google Search | 99.99% | 200ms | 0.1% |
| AWS S3 | 99.9% | 500ms | 0.1% |
| Stripe API | 99.99% | 500ms | 0.01% |
| Slack | 99.9% | 1000ms | 0.5% |

### Шаг 4: Определить SLA

```yaml
sla_with_compensation:
  service: "Payment Gateway API"
  period: "monthly"

  service_credits:
    - availability_range: "99.0% - 99.9%"
      credit_percent: 10
    - availability_range: "95.0% - 99.0%"
      credit_percent: 25
    - availability_range: "< 95.0%"
      credit_percent: 50
      escalation: true

  exclusions:
    - maintenance_windows: "scheduled"
    - force_majeure: true
    - customer_actions: true
    - beta_features: true
```

---

## Расчет Error Budget и Burn Rate

### Error Budget

```python
class ErrorBudgetCalculator:
    def __init__(self, slo_target, window_days=28):
        self.slo_target = slo_target  # 0.999 for 99.9%
        self.window_days = window_days
        self.total_minutes = window_days * 24 * 60

    def calculate_budget(self):
        """Вычисляем допустимое количество плохих минут"""
        return (1 - self.slo_target) * self.total_minutes

    def calculate_burn_rate(self, bad_minutes, window_hours=1):
        """Вычисляем скорость сжигания бюджета"""
        budget = self.calculate_budget()
        current_burn = bad_minutes / (window_hours * 60)
        return current_burn / budget

    def get_slo_status(self, bad_minutes):
        """Получаем статус SLO"""
        budget = self.calculate_budget()
        remaining = budget - bad_minutes
        consumed = (bad_minutes / budget) * 100

        return {
            'budget_total_minutes': budget,
            'budget_minutes_used': bad_minutes,
            'budget_minutes_remaining': remaining,
            'budget_percent_consumed': consumed,
            'budget_percent_remaining': 100 - consumed
        }


# Пример
slo_999 = ErrorBudgetCalculator(slo_target=0.999)
status = slo_999.get_slo_status(bad_minutes=20)

# Output
{
    'budget_total_minutes': 40.32,  # 28 дней × 24 часа × 60 минут × 0.1%
    'budget_minutes_used': 20,
    'budget_minutes_remaining': 20.32,
    'budget_percent_consumed': 49.6,
    'budget_percent_remaining': 50.4
}
```

### Burn Rate Alerts

```yaml
# Правила алертинга на основе скорости сжигания бюджета
alerting_rules:
  critical_fast_burn:
    burn_rate: 14.4  # Сожжем весь бюджет за 2 часа
    window: 1h
    severity: critical
    action: page_oncall

  critical_medium_burn:
    burn_rate: 6  # Сожжем за 4 часа
    window: 6h
    severity: warning
    action: create_incident

  warning_slow_burn:
    burn_rate: 2  # Сожжем за 36 часов
    window: 24h
    severity: info
    action: notify_team
```

---

## SLI/SLO Dashboards

### API Dashboard (Grafana)

```json
{
  "dashboard": {
    "title": "Payment Service SLO",
    "panels": [
      {
        "title": "Availability (Last 28 Days)",
        "type": "gauge",
        "targets": [{
          "expr": "sum(rate(http_requests_success[28d])) / sum(rate(http_requests_total[28d]))"
        }],
        "thresholds": {
          "green": 0.999,
          "yellow": 0.995,
          "red": 0.99
        }
      },
      {
        "title": "Error Budget Remaining",
        "type": "stat",
        "targets": [{
          "expr": "(1 - (sum(rate(http_requests_5xx[28d])) / sum(rate(http_requests_total[28d])))) - 0.999"
        }],
        "unit": "percent"
      },
      {
        "title": "Latency P99",
        "type": "graph",
        "targets": [
          {"expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))"},
          {"expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))"},
          {"expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))"}
        ]
      },
      {
        "title": "Burn Rate (Last 6h)",
        "type": "stat",
        "targets": [{
          "expr": "sum(rate(http_requests_5xx[6h])) / sum(rate(http_requests_total[6h])) / (1 - 0.999)"
        }],
        "thresholds": {
          "green": 0,
          "red": 6
        }
      }
    ]
  }
}
```

---

## SLO Review Process

### Ежеквартальный SLO Review

```yaml
slo_review_process:
  frequency: quarterly
  attendees: [sre_team, product_team, engineering_leads]

  agenda:
    1_slo_performance:
      - текущие значения SLI
      - процент использования error budget
      - incidents и their impact

    2_trend_analysis:
      - тренды за последние 3 месяца
      - сезонные паттерны
      - корреляция с releases

    3_error_budget_policy:
      - сколько budget осталось
      - frozen or normal mode
      - feature freeze decisions

    4_action_items:
      - улучшения мониторинга
      - reliability project prioritization
      - SLO adjustments
```

### Error Budget Policy

```yaml
error_budget_policy:
  budget_states:
    healthy:
      consumed: "< 50%"
      mode: normal
      actions: business_as_usual

    at_risk:
      consumed: "50% - 70%"
      mode: cautious
      actions:
        - limit_new_features
        - no_risky_deploys
        - increase_testing

    depleted:
      consumed: "> 70%"
      mode: emergency
      actions:
        - feature_freeze
        - focus_on_reliability
        - oncall_workload_100%
        - no_deploys_without_approval

    over_budget:
      consumed: "> 100%"
      mode: emergency
      actions:
        - all_hands_on_deck
        - stop_all_deploys
        - revise_slo
```

---

## Примеры SLI/SLO/SLA по сервисам

### E-commerce API

```yaml
service_tiers:
  checkout_api:
    tier: critical
    slos:
      availability: 99.99%  # 4.3 минуты в месяц
      latency_p99: 500ms
      error_rate: 0.01%
    sla:
      - < 99.99%: 10% credit
      - < 99.9%: 25% credit
      - < 99%: 50% credit

  product_catalog:
    tier: important
    slos:
      availability: 99.9%
      latency_p99: 1000ms
      error_rate: 0.1%
```

### Data Pipeline

```yaml
pipeline_tiers:
  data_ingestion:
    tier: critical
    slos:
      freshness: 5 minutes
      completeness: 99.9%
      correctness: 99.99%

  data_processing:
    tier: important
    slos:
      freshness: 1 hour
      completeness: 99%
      correctness: 99.9%

  data_analytics:
    tier: best_effort
    slos:
      freshness: 24 hours
      completeness: 95%
```

### Mobile App Backend

```yaml
mobile_backend:
  tier: critical
  slos:
    availability: 99.95%
    latency_p99: 200ms
    latency_p95: 100ms
    push_notification_delivery: 99.5%
```

---

## Использование Error Budget для принятия решений

```python
class SLOBasedDecisionMaking:
    def __init__(self, slo_calculator):
        self.slo = slo_calculator

    def can_deploy_new_features(self):
        """Можно ли деплоить новые фичи"""
        status = self.slo.get_slo_status()

        if status['budget_percent_remaining'] > 70:
            return True, "SLO healthy, proceed with feature deployment"

        elif status['budget_percent_remaining'] > 50:
            return True, "SLO at risk, deploy cautiously with extra testing"

        else:
            return False, "SLO depleted, focus on reliability improvements"

    def should_invest_in_reliability(self):
        """Нужно ли инвестировать в надежность"""
        status = self.slo.get_slo_status()

        if status['budget_percent_consumed'] > 70:
            return True, f"High reliability investment needed. Budget consumed: {status['budget_percent_consumed']:.1f}%"

        elif status['budget_percent_consumed'] > 50:
            return True, f"Moderate reliability improvements recommended"

        else:
            return False, "SLO healthy, maintain current reliability level"

    def get_recommendations(self):
        """Получить рекомендации"""
        deploy_features, msg1 = self.can_deploy_new_features()
        invest_reliability, msg2 = self.should_invest_in_reliability()

        return {
            'deploy_new_features': deploy_features,
            'invest_in_reliability': invest_reliability,
            'recommendations': [msg1, msg2]
        }
```

---

## Сравнение SLA по индустрии

| Компания | Услуга | Доступность | Latency | Компенсация |
|----------|--------|-------------|---------|-------------|
| AWS EC2 | Базовый | 99.5% | - | 10% credit |
| AWS S3 | Standard | 99.9% | - | 10-25% credit |
| GCP Cloud Storage | Multi-regional | 99.95% | - | 10-50% credit |
| Azure Storage | Standard | 99.9% | - | 10-25% credit |
| Stripe | Payments | 99.99% | 200ms P99 | Varies |
| Twilio | SMS | 99.95% | - | $1-10 credit |
| SendGrid | Email | 99.99% | 5s | 10-100% credit |
| Cloudflare | Business | 100% | - | 25-100% credit |
| Datadog | Pro | 99.95% | - | 10-100% credit |

---

## Метрики SRE

### Four Golden Signals (Четыре золотых сигнала)

```yaml
golden_signals:
  latency:
    description: "Время обработки запросов"
    slis: [p50, p95, p99]

  traffic:
    description: "Количество запросов"
    slis: [requests_per_second, active_connections]

  errors:
    description: "Доля ошибок"
    slis: [error_rate, error_count_by_code]

  saturation:
    description: "Загрузка системы"
    slis: [cpu_percent, memory_percent, disk_io]
```

### RED Method (Rate, Errors, Duration)

```yaml
red_method:
  rate:
    description: "Запросов в секунду"
    formula: 'rate(http_requests_total[5m])'

  errors:
    description: "Ошибочных запросов"
    formula: 'rate(http_requests_5xx[5m])'

  duration:
    description: "Время обработки"
    formula: 'histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))'
```

### USE Method (Utilization, Saturation, Errors)

```yaml
use_method:
  utilization:
    description: "Использование ресурса"
    examples: ['cpu_percent', 'memory_percent', 'disk_io_percent']

  saturation:
    description: "Очередь на ресурс"
    examples: ['load_average', 'queue_length', 'network_buffer']

  errors:
    description: "Ошибки ресурса"
    examples: ['network_errors', 'disk_errors', 'memory_failures']
```

---

## SLI/SLO Dashboard на Grafana

```json
{
  "dashboard": {
    "title": "SRE SLI/SLO Dashboard",
    "panels": [
      {
        "title": "SLO Status Overview",
        "type": "stat",
        "targets": [{
          "expr": "slo:status:availability"
        }],
        "thresholds": {
          "green": 0.999,
          "yellow": 0.995,
          "red": 0.99
        }
      },
      {
        "title": "Error Budget Burn Rate",
        "type": "graph",
        "targets": [{
          "expr": "slo:error_budget_burn_rate"
        }]
      },
      {
        "title": "SLI Trends (28 days)",
        "type": "timeseries",
        "targets": [
          {"expr": "sli:availability:28d"},
          {"expr": "sli:latency:p99:28d"},
          {"expr": "sli:error_rate:28d"}
        ]
      },
      {
        "title": "Top Burn Rate Incidents",
        "type": "table",
        "targets": [{
          "expr": "topk(10, slo:incidents:by_budget_consumed)"
        }]
      }
    ]
  }
}
```

---

*SLI/SLO/SLA - фреймворк для определения и управления надежностью сервисов*
