# SRO-003 Cost Optimization & FinOps

FinOps (Cloud Financial Operations) — a practice of managing cloud costs that brings together finance, technology, and business to make data-driven decisions.

## Why FinOps?

### Problems Without FinOps
- Uncontrolled cloud spending growth
- No accountability for costs
- Inability to forecast budget
- "Bill shock" at month end

### FinOps Benefits
- **Visibility**: understanding who spends on what
- **Optimization**: 20-40% cost reduction
- **Predictability**: accurate budget planning
- **Accountability**: teams own their spending

## FinOps Principles

### 1. Teams Own Their Costs
```
Traditional approach:
  IT Department → pays for everything → no accountability

FinOps approach:
  Team A → sees their costs → optimizes
  Team B → sees their costs → optimizes
  Platform → shared infrastructure → allocates
```

### 2. Decisions Based on Business Value
```
Not just "how much does it cost?" but "what's the ROI?"

Example:
  Service A: $10,000/mo → generates $100,000 revenue → ROI 10x ✅
  Service B: $5,000/mo → generates $2,000 revenue → ROI 0.4x ❌
```

### 3. Centralized FinOps Team
```
FinOps Team:
├── Cloud Economist (financial analyst)
├── FinOps Engineer (technical specialist)
└── FinOps Practitioner (coordinator)

Responsibilities:
- Creating dashboards and reports
- Identifying optimization opportunities
- Training teams
- Vendor negotiations
```

## Tagging Strategy

### Required Tags

```yaml
# Minimum tag set
required_tags:
  - key: "team"
    description: "Team that owns the resource"
    examples: ["platform", "payments", "search"]

  - key: "service"
    description: "Service name"
    examples: ["api-gateway", "user-service", "analytics"]

  - key: "environment"
    description: "Environment"
    examples: ["production", "staging", "development"]

  - key: "cost-center"
    description: "Cost center for accounting"
    examples: ["CC-001", "CC-002"]
```

### Optional Tags

```yaml
optional_tags:
  - key: "project"
    description: "Project or initiative"

  - key: "owner"
    description: "Owner's email"

  - key: "expiry-date"
    description: "Deletion date (for temporary resources)"

  - key: "data-classification"
    description: "Data classification"
    examples: ["public", "internal", "confidential"]
```

### Tagging Automation

```python
# Terraform: enforce tagging
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

# Kubernetes: automatic labels
# kustomization.yaml
commonLabels:
  team: payments
  service: payment-gateway
  environment: production
```

### Tagging Policy (AWS)

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

### Allocation Models

#### 1. Direct Allocation
```
Resource belongs to one team → 100% cost to that team

Example:
  EC2 instance (tag: team=payments) → $100 → Team Payments
```

#### 2. Shared Costs
```
Shared resources distributed by formula:

Distribution options:
├── By usage (CPU hours, requests)
├── By headcount (number of developers)
├── Evenly (equal split between teams)
└── By revenue (revenue share)

Example (by usage):
  Kubernetes cluster: $10,000/mo
  Team A: 60% CPU → $6,000
  Team B: 30% CPU → $3,000
  Team C: 10% CPU → $1,000
```

#### 3. Showback vs Chargeback
```
Showback:
  - Show costs to teams
  - No actual billing
  - For increasing awareness

Chargeback:
  - Actual billing to team budget
  - Requires accurate accounting
  - Maximum accountability
```

### Kubernetes Cost Allocation

```yaml
# Kubecost / OpenCost configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubecost-config
data:
  # Shared costs distribution
  sharedNamespaces: "kube-system,monitoring,istio-system"
  sharedOverhead: "0.2"  # 20% overhead

  # Labels for allocation
  allocationLabels: "team,service,environment"
```

```python
# Prometheus queries for cost allocation
queries:
  # CPU cost by namespace
  cpu_cost_by_namespace: |
    sum(
      rate(container_cpu_usage_seconds_total[1h])
      * on(node) group_left() node_cpu_hourly_cost
    ) by (namespace)

  # Memory cost by namespace
  memory_cost_by_namespace: |
    sum(
      container_memory_working_set_bytes
      * on(node) group_left() node_ram_hourly_cost / 1024 / 1024 / 1024
    ) by (namespace)
```

## Cost Optimization

### 1. Right-sizing

```python
# Resource utilization analysis
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

**Right-sizing recommendations:**
```
CPU utilization:
  < 20% avg, < 50% max → Downsize
  > 80% avg → Upsize or auto-scaling

Memory utilization:
  < 30% avg → Downsize
  > 85% avg → Upsize

Check every 2 weeks
```

### 2. Reserved Instances / Savings Plans

```yaml
# Reserved Instances purchase strategy
ri_strategy:
  # Baseline load - covered by RI
  baseline_coverage: 70%
  commitment_term: "1-year"  # or "3-year" for bigger discount
  payment_option: "partial-upfront"  # balance of discount and risk

  # Peak load - On-Demand or Spot
  peak_coverage: "on-demand"

# Savings by type:
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
# Workloads for Spot
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

**When to use Spot:**
```
✅ Good for Spot:
- Batch processing
- CI/CD runners
- Dev/Test environments
- Stateless workers
- ML training

❌ Not good for Spot:
- Databases
- Stateful services
- Real-time critical services
```

### 4. Automatic Shutdown

```python
# Lambda to stop dev/staging resources
import boto3

def stop_non_production_instances(event, context):
    ec2 = boto3.client('ec2')

    # Find dev/staging instances
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

# CloudWatch Event Rule: cron(0 20 ? * MON-FRI *)  # 20:00 weekdays
# CloudWatch Event Rule: cron(0 8 ? * MON-FRI *)   # 08:00 start
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
      days: 2555  # 7 years

# EBS optimization
ebs_recommendations:
  - check: "unattached_volumes"
    action: "delete or snapshot"

  - check: "gp2_to_gp3"
    action: "migrate"  # GP3 is cheaper and faster

  - check: "old_snapshots"
    action: "delete snapshots older than 90 days"
```

## Cost Anomaly Detection

### Alert Configuration

```yaml
# AWS Cost Anomaly Detection
anomaly_detection:
  monitors:
    - name: "service-monitor"
      type: "DIMENSIONAL"
      dimension: "SERVICE"
      threshold: 20  # % of expected

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

### Prometheus Alerts for Costs

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
# Cost forecasting
from prophet import Prophet
import pandas as pd

def forecast_costs(historical_data: pd.DataFrame, periods: int = 30):
    """
    Forecast costs based on historical data.

    historical_data: DataFrame with columns 'ds' (date) and 'y' (cost)
    periods: number of days to forecast
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

## Tools

### Platform Comparison

| Tool | Type | Kubernetes | Multi-cloud | Price |
|------|------|------------|-------------|-------|
| AWS Cost Explorer | Native | No | No | Free |
| Kubecost | OSS/Enterprise | Yes | Yes | Free tier |
| CloudHealth | SaaS | Yes | Yes | $$$ |
| Spot.io | SaaS | Yes | Yes | $$ |
| Infracost | OSS | No | Yes | Free tier |

### Kubecost Setup

```bash
# Install Kubecost
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost \
  --create-namespace \
  --set kubecostToken="YOUR_TOKEN"
```

### Infracost (for Terraform)

```yaml
# GitHub Actions: cost estimation in PR
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

### Level 1: Crawl (Beginning)
```
✅ Basic cost visibility
✅ Minimal tagging
✅ Monthly reports
❌ No allocation
❌ No optimization
```

### Level 2: Walk (Developing)
```
✅ Complete tagging
✅ Cost allocation by team
✅ Showback reports
✅ Basic optimizations (right-sizing)
✅ Weekly reviews
❌ No forecasting
❌ No automation
```

### Level 3: Run (Optimized)
```
✅ Automated chargeback
✅ Real-time anomaly detection
✅ Forecasting and budgeting
✅ Automated optimization
✅ Unit economics (cost per transaction)
✅ FinOps culture in teams
```

## Best Practices

### 1. Start with Visibility
```
Week 1-2: Set up cost explorer and basic dashboards
Week 3-4: Implement required tags
Month 2: Showback reports to teams
Month 3: First optimizations
```

### 2. Create FinOps Rituals
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
# Unit economics metrics
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

## Anti-patterns

### 1. Cost Cutting Without Context
```
❌ "Let's downsize all instances by one size"
✅ "Let's analyze utilization and right-size precisely"
```

### 2. Ignoring Shared Costs
```
❌ Shared costs = "someone else pays"
✅ Fair distribution by usage
```

### 3. Only Monthly Reviews
```
❌ Learning about problems at month end
✅ Real-time monitoring and anomaly detection
```

### 4. Lack of Accountability
```
❌ "Cloud is IT's problem"
✅ Each team owns their costs
```

## References

- [FinOps Foundation](https://www.finops.org/)
- [AWS Cost Management](https://aws.amazon.com/aws-cost-management/)
- [Kubecost Documentation](https://docs.kubecost.com/)
- [Cloud Cost Optimization - Google](https://cloud.google.com/architecture/cost-optimization)
- [FinOps Certified Practitioner](https://learn.finops.org/)
