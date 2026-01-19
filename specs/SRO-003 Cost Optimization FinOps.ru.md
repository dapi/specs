# SRO-003 Cost Optimization & FinOps

FinOps (Cloud Financial Operations) — практика управления облачными затратами, объединяющая финансы, технологии и бизнес для принятия решений на основе данных.

## Зачем нужен FinOps?

### Проблемы без FinOps
- Неконтролируемый рост облачных расходов
- Отсутствие ответственных за затраты
- Невозможность прогнозирования бюджета
- "Bill shock" в конце месяца

### Преимущества FinOps
- **Видимость**: понимание кто и на что тратит
- **Оптимизация**: снижение затрат на 20-40%
- **Прогнозируемость**: точное планирование бюджета
- **Accountability**: команды отвечают за свои расходы

## Принципы FinOps

### 1. Команды владеют своими расходами
```
Традиционный подход:
  IT Department → платит за всё → нет accountability

FinOps подход:
  Team A → видит свои затраты → оптимизирует
  Team B → видит свои затраты → оптимизирует
  Platform → общая инфраструктура → распределяет
```

### 2. Решения на основе бизнес-ценности
```
Не просто "сколько стоит?" а "какой ROI?"

Пример:
  Сервис A: $10,000/мес → генерирует $100,000 revenue → ROI 10x ✅
  Сервис B: $5,000/мес → генерирует $2,000 revenue → ROI 0.4x ❌
```

### 3. Централизованная команда FinOps
```
FinOps Team:
├── Cloud Economist (финансовый аналитик)
├── FinOps Engineer (технический специалист)
└── FinOps Practitioner (координатор)

Обязанности:
- Создание дашбордов и отчётов
- Выявление возможностей оптимизации
- Обучение команд
- Переговоры с вендорами
```

## Tagging Strategy

### Обязательные теги

```yaml
# Минимальный набор тегов
required_tags:
  - key: "team"
    description: "Команда-владелец ресурса"
    examples: ["platform", "payments", "search"]

  - key: "service"
    description: "Название сервиса"
    examples: ["api-gateway", "user-service", "analytics"]

  - key: "environment"
    description: "Окружение"
    examples: ["production", "staging", "development"]

  - key: "cost-center"
    description: "Центр затрат для бухгалтерии"
    examples: ["CC-001", "CC-002"]
```

### Дополнительные теги

```yaml
optional_tags:
  - key: "project"
    description: "Проект или инициатива"

  - key: "owner"
    description: "Email ответственного"

  - key: "expiry-date"
    description: "Дата удаления (для временных ресурсов)"

  - key: "data-classification"
    description: "Классификация данных"
    examples: ["public", "internal", "confidential"]
```

### Автоматизация тегирования

```python
# Terraform: принудительное тегирование
# providers.tf
provider "aws" {
  default_tags {
    tags = {
      Team        = var.team
      Service     = var.service
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}

# Kubernetes: автоматические labels
# kustomization.yaml
commonLabels:
  team: payments
  service: payment-gateway
  environment: production
```

### Политика тегирования (AWS)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RequireTags",
      "Effect": "Deny",
      "Action": [
        "ec2:RunInstances",
        "rds:CreateDBInstance"
      ],
      "Resource": "*",
      "Condition": {
        "Null": {
          "aws:RequestTag/team": "true",
          "aws:RequestTag/service": "true",
          "aws:RequestTag/environment": "true"
        }
      }
    }
  ]
}
```

## Cost Allocation

### Модели распределения затрат

#### 1. Direct Allocation (Прямое)
```
Ресурс принадлежит одной команде → 100% затрат этой команде

Пример:
  EC2 instance (tag: team=payments) → $100 → Team Payments
```

#### 2. Shared Costs (Общие затраты)
```
Общие ресурсы распределяются по формуле:

Варианты распределения:
├── По использованию (CPU hours, requests)
├── По headcount (количество разработчиков)
├── Равномерно (поровну между командами)
└── По revenue (доле дохода)

Пример (по использованию):
  Kubernetes cluster: $10,000/мес
  Team A: 60% CPU → $6,000
  Team B: 30% CPU → $3,000
  Team C: 10% CPU → $1,000
```

#### 3. Showback vs Chargeback
```
Showback:
  - Показываем затраты командам
  - Нет реального списания
  - Для повышения awareness

Chargeback:
  - Реальное списание с бюджета команды
  - Требует точного учёта
  - Максимальная accountability
```

### Kubernetes Cost Allocation

```yaml
# Kubecost / OpenCost конфигурация
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubecost-config
data:
  # Распределение shared costs
  sharedNamespaces: "kube-system,monitoring,istio-system"
  sharedOverhead: "0.2"  # 20% overhead

  # Labels для аллокации
  allocationLabels: "team,service,environment"
```

```python
# Prometheus запросы для cost allocation
queries:
  # CPU cost по namespace
  cpu_cost_by_namespace: |
    sum(
      rate(container_cpu_usage_seconds_total[1h])
      * on(node) group_left() node_cpu_hourly_cost
    ) by (namespace)

  # Memory cost по namespace
  memory_cost_by_namespace: |
    sum(
      container_memory_working_set_bytes
      * on(node) group_left() node_ram_hourly_cost / 1024 / 1024 / 1024
    ) by (namespace)
```

## Оптимизация затрат

### 1. Right-sizing

```python
# Анализ использования ресурсов
class RightSizingAnalyzer:
    def analyze_instance(self, instance_id: str) -> dict:
        metrics = self.get_cloudwatch_metrics(instance_id, days=14)

        return {
            "current_type": "m5.xlarge",
            "cpu_avg": metrics["cpu_avg"],  # 15%
            "cpu_max": metrics["cpu_max"],  # 45%
            "memory_avg": metrics["memory_avg"],  # 20%
            "recommendation": "m5.large",  # Downsize
            "monthly_savings": 150.00
        }
```

**Рекомендации по right-sizing:**
```
CPU utilization:
  < 20% avg, < 50% max → Downsize
  > 80% avg → Upsize или auto-scaling

Memory utilization:
  < 30% avg → Downsize
  > 85% avg → Upsize

Проверять каждые 2 недели
```

### 2. Reserved Instances / Savings Plans

```yaml
# Стратегия покупки Reserved Instances
ri_strategy:
  # Базовая нагрузка - покрывается RI
  baseline_coverage: 70%
  commitment_term: "1-year"  # или "3-year" для большей скидки
  payment_option: "partial-upfront"  # баланс скидки и риска

  # Пиковая нагрузка - On-Demand или Spot
  peak_coverage: "on-demand"

# Экономия по типам:
# 1-year No Upfront: ~30% savings
# 1-year Partial Upfront: ~40% savings
# 3-year All Upfront: ~60% savings
```

### 3. Spot Instances

```yaml
# Kubernetes Spot instances
apiVersion: v1
kind: Node
metadata:
  labels:
    node.kubernetes.io/lifecycle: spot
---
# Workloads для Spot
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-processor
spec:
  template:
    spec:
      nodeSelector:
        node.kubernetes.io/lifecycle: spot
      tolerations:
        - key: "spot"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
```

**Когда использовать Spot:**
```
✅ Подходит для Spot:
- Batch processing
- CI/CD runners
- Dev/Test environments
- Stateless workers
- ML training

❌ Не подходит для Spot:
- Databases
- Stateful services
- Real-time критичные сервисы
```

### 4. Автоматическое выключение

```python
# Lambda для выключения dev/staging ресурсов
import boto3

def stop_non_production_instances(event, context):
    ec2 = boto3.client('ec2')

    # Находим dev/staging instances
    response = ec2.describe_instances(
        Filters=[
            {'Name': 'tag:environment', 'Values': ['development', 'staging']},
            {'Name': 'instance-state-name', 'Values': ['running']}
        ]
    )

    instance_ids = [
        i['InstanceId']
        for r in response['Reservations']
        for i in r['Instances']
    ]

    if instance_ids:
        ec2.stop_instances(InstanceIds=instance_ids)
        return f"Stopped {len(instance_ids)} instances"

    return "No instances to stop"

# CloudWatch Event Rule: cron(0 20 ? * MON-FRI *)  # 20:00 будни
# CloudWatch Event Rule: cron(0 8 ? * MON-FRI *)   # 08:00 старт
```

### 5. Storage Optimization

```yaml
# S3 Lifecycle Policy
lifecycle_rules:
  - id: "move-to-glacier"
    status: "Enabled"
    transitions:
      - days: 30
        storage_class: "STANDARD_IA"
      - days: 90
        storage_class: "GLACIER"
      - days: 365
        storage_class: "DEEP_ARCHIVE"
    expiration:
      days: 2555  # 7 лет

# EBS optimization
ebs_recommendations:
  - check: "unattached_volumes"
    action: "delete or snapshot"

  - check: "gp2_to_gp3"
    action: "migrate"  # GP3 дешевле и быстрее

  - check: "old_snapshots"
    action: "delete snapshots older than 90 days"
```

## Cost Anomaly Detection

### Настройка алертов

```yaml
# AWS Cost Anomaly Detection
anomaly_detection:
  monitors:
    - name: "service-monitor"
      type: "DIMENSIONAL"
      dimension: "SERVICE"
      threshold: 20  # % от ожидаемого

    - name: "team-monitor"
      type: "COST_ALLOCATION_TAG"
      tag_key: "team"
      threshold: 30

  alerts:
    - threshold_type: "PERCENTAGE"
      threshold: 20
      notification:
        type: "SNS"
        topic_arn: "arn:aws:sns:us-east-1:123456789:cost-alerts"
```

### Prometheus алерты для затрат

```yaml
groups:
  - name: cost-alerts
    rules:
      - alert: CostAnomalyDetected
        expr: |
          (
            sum(kubecost_cluster_cost_total)
            - sum(kubecost_cluster_cost_total offset 7d)
          ) / sum(kubecost_cluster_cost_total offset 7d) > 0.2
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Cost increased by more than 20% week-over-week"

      - alert: BudgetExceeded
        expr: |
          sum(kubecost_cluster_cost_total) > 50000
        for: 1h
        labels:
          severity: critical
        annotations:
          summary: "Monthly budget exceeded"
```

## Budgeting & Forecasting

### AWS Budgets

```json
{
  "BudgetName": "monthly-team-payments",
  "BudgetLimit": {
    "Amount": "10000",
    "Unit": "USD"
  },
  "BudgetType": "COST",
  "TimeUnit": "MONTHLY",
  "CostFilters": {
    "TagKeyValue": ["user:team$payments"]
  },
  "NotificationsWithSubscribers": [
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [
        {
          "SubscriptionType": "EMAIL",
          "Address": "payments-team@company.com"
        }
      ]
    },
    {
      "Notification": {
        "NotificationType": "FORECASTED",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 100,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [
        {
          "SubscriptionType": "SNS",
          "Address": "arn:aws:sns:us-east-1:123456789:budget-alerts"
        }
      ]
    }
  ]
}
```

### Forecasting

```python
# Прогнозирование затрат
from prophet import Prophet
import pandas as pd

def forecast_costs(historical_data: pd.DataFrame, periods: int = 30):
    """
    Прогноз затрат на основе исторических данных.

    historical_data: DataFrame с колонками 'ds' (date) и 'y' (cost)
    periods: количество дней для прогноза
    """
    model = Prophet(
        yearly_seasonality=True,
        weekly_seasonality=True,
        daily_seasonality=False
    )
    model.fit(historical_data)

    future = model.make_future_dataframe(periods=periods)
    forecast = model.predict(future)

    return {
        "next_month_forecast": forecast['yhat'].tail(30).sum(),
        "confidence_interval": {
            "lower": forecast['yhat_lower'].tail(30).sum(),
            "upper": forecast['yhat_upper'].tail(30).sum()
        }
    }
```

## Инструменты

### Сравнение платформ

| Инструмент | Тип | Kubernetes | Multi-cloud | Цена |
|------------|-----|------------|-------------|------|
| AWS Cost Explorer | Native | Нет | Нет | Бесплатно |
| Kubecost | OSS/Enterprise | Да | Да | Free tier |
| CloudHealth | SaaS | Да | Да | $$$ |
| Spot.io | SaaS | Да | Да | $$ |
| Infracost | OSS | Нет | Да | Free tier |

### Kubecost Setup

```bash
# Установка Kubecost
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost \
  --create-namespace \
  --set kubecostToken="YOUR_TOKEN"
```

### Infracost (для Terraform)

```yaml
# GitHub Actions: cost estimation в PR
name: Infracost
on: [pull_request]

jobs:
  infracost:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Infracost
        uses: infracost/actions/setup@v2
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}

      - name: Generate cost estimate
        run: |
          infracost breakdown --path=. \
            --format=json \
            --out-file=/tmp/infracost.json

      - name: Post PR comment
        uses: infracost/actions/comment@v1
        with:
          path: /tmp/infracost.json
          behavior: update
```

## FinOps Maturity Model

### Level 1: Crawl (Начальный)
```
✅ Базовая видимость затрат
✅ Минимальное тегирование
✅ Ежемесячные отчёты
❌ Нет allocation
❌ Нет оптимизации
```

### Level 2: Walk (Развивающийся)
```
✅ Полное тегирование
✅ Cost allocation по командам
✅ Showback отчёты
✅ Базовые оптимизации (right-sizing)
✅ Еженедельные reviews
❌ Нет forecasting
❌ Нет automation
```

### Level 3: Run (Оптимизированный)
```
✅ Автоматический chargeback
✅ Real-time anomaly detection
✅ Forecasting и budgeting
✅ Automated optimization
✅ Unit economics (cost per transaction)
✅ FinOps culture в командах
```

## Best Practices

### 1. Начните с visibility
```
Week 1-2: Настройка cost explorer и базовых дашбордов
Week 3-4: Внедрение обязательных тегов
Month 2: Showback отчёты командам
Month 3: Первые оптимизации
```

### 2. Создайте FinOps ритуалы
```yaml
finops_rituals:
  daily:
    - check_anomaly_alerts

  weekly:
    - team_cost_review
    - optimization_opportunities

  monthly:
    - budget_vs_actual_review
    - reserved_instance_planning
    - forecast_update

  quarterly:
    - vendor_negotiations
    - commitment_review
```

### 3. Unit Economics
```python
# Метрики unit economics
metrics:
  cost_per_request:
    formula: "total_cost / total_requests"
    target: "$0.0001"

  cost_per_user:
    formula: "total_cost / monthly_active_users"
    target: "$0.50"

  cost_per_transaction:
    formula: "total_cost / successful_transactions"
    target: "$0.05"
```

## Антипаттерны

### 1. Cost cutting без контекста
```
❌ "Уменьшим все instances на один размер"
✅ "Проанализируем utilization и right-size точечно"
```

### 2. Игнорирование shared costs
```
❌ Shared costs = "кто-то другой платит"
✅ Справедливое распределение по использованию
```

### 3. Только monthly reviews
```
❌ Узнаём о проблемах в конце месяца
✅ Real-time мониторинг и anomaly detection
```

### 4. Отсутствие accountability
```
❌ "Облако — это проблема IT"
✅ Каждая команда отвечает за свои затраты
```

## Ссылки

- [FinOps Foundation](https://www.finops.org/)
- [AWS Cost Management](https://aws.amazon.com/aws-cost-management/)
- [Kubecost Documentation](https://docs.kubecost.com/)
- [Cloud Cost Optimization - Google](https://cloud.google.com/architecture/cost-optimization)
- [FinOps Certified Practitioner](https://learn.finops.org/)
