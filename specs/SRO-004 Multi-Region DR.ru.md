# SRO-004 Multi-Region DR

Multi-Region DR — стратегия обеспечения непрерывности бизнеса путём распределения инфраструктуры по нескольким географическим регионам.

## Зачем Multi-Region?

### Проблемы Single-Region
- Полный отказ при региональном сбое
- Высокая latency для удалённых пользователей
- Невозможность выполнить требования compliance
- Риск потери данных при катастрофе

### Преимущества Multi-Region
- **Отказоустойчивость**: продолжение работы при отказе региона
- **Низкая latency**: обслуживание пользователей из ближайшего региона
- **Compliance**: данные хранятся в требуемых юрисдикциях
- **Масштабируемость**: распределение нагрузки по регионам

## RTO и RPO

### Определения

```
RTO (Recovery Time Objective):
  Максимально допустимое время восстановления после сбоя
  "Как долго мы можем быть недоступны?"

RPO (Recovery Point Objective):
  Максимально допустимый объём потери данных
  "Сколько данных мы можем потерять?"

Пример:
  RTO = 1 час → система должна восстановиться за 1 час
  RPO = 15 минут → допустима потеря данных за последние 15 минут
```

### Классификация систем

```yaml
tier_classification:
  tier_1_critical:
    description: "Критичные для бизнеса системы"
    examples: ["платежи", "авторизация", "основной API"]
    rto: "< 15 минут"
    rpo: "< 1 минута"
    strategy: "Active-Active"

  tier_2_important:
    description: "Важные системы"
    examples: ["поиск", "рекомендации", "аналитика"]
    rto: "< 1 час"
    rpo: "< 15 минут"
    strategy: "Active-Passive Hot"

  tier_3_standard:
    description: "Стандартные системы"
    examples: ["отчёты", "внутренние инструменты"]
    rto: "< 4 часа"
    rpo: "< 1 час"
    strategy: "Active-Passive Warm"

  tier_4_low:
    description: "Некритичные системы"
    examples: ["dev/staging", "архивы"]
    rto: "< 24 часа"
    rpo: "< 24 часа"
    strategy: "Backup/Restore"
```

## Архитектурные паттерны

### 1. Active-Passive (Hot Standby)

```
┌─────────────────┐         ┌─────────────────┐
│  Primary Region │         │ Secondary Region│
│    (Active)     │         │   (Passive)     │
├─────────────────┤         ├─────────────────┤
│   ┌─────────┐   │         │   ┌─────────┐   │
│   │   App   │   │         │   │   App   │   │
│   │ Servers │   │         │   │ Servers │   │
│   └────┬────┘   │         │   └────┬────┘   │
│        │        │         │        │        │
│   ┌────▼────┐   │  Sync   │   ┌────▼────┐   │
│   │Database │───┼────────►│   │Database │   │
│   │ Primary │   │         │   │ Replica │   │
│   └─────────┘   │         │   └─────────┘   │
└─────────────────┘         └─────────────────┘
        │                           │
        └───────────┬───────────────┘
                    │
              ┌─────▼─────┐
              │   Global  │
              │Load Balancer│
              └───────────┘

Характеристики:
- Один регион обрабатывает весь трафик
- Второй регион готов к переключению
- RTO: минуты - часы
- Стоимость: ~1.5x от single-region
```

### 2. Active-Active

```
┌─────────────────┐         ┌─────────────────┐
│    Region A     │         │    Region B     │
│    (Active)     │         │    (Active)     │
├─────────────────┤         ├─────────────────┤
│   ┌─────────┐   │         │   ┌─────────┐   │
│   │   App   │   │         │   │   App   │   │
│   │ Servers │   │         │   │ Servers │   │
│   └────┬────┘   │         │   └────┬────┘   │
│        │        │         │        │        │
│   ┌────▼────┐   │◄───────►│   ┌────▼────┐   │
│   │Database │   │  Sync   │   │Database │   │
│   │ Multi-  │   │         │   │ Multi-  │   │
│   │ Master  │   │         │   │ Master  │   │
│   └─────────┘   │         │   └─────────┘   │
└─────────────────┘         └─────────────────┘
        │                           │
        └───────────┬───────────────┘
              ┌─────▼─────┐
              │   Global  │
              │    DNS    │
              │(GeoDNS/GLB)│
              └───────────┘

Характеристики:
- Оба региона обрабатывают трафик
- Автоматическое распределение по географии
- RTO: секунды
- Стоимость: ~2x от single-region
```

### 3. Pilot Light

```
┌─────────────────┐         ┌─────────────────┐
│  Primary Region │         │    DR Region    │
│    (Active)     │         │  (Pilot Light)  │
├─────────────────┤         ├─────────────────┤
│   ┌─────────┐   │         │                 │
│   │Full Stack│   │         │  Минимальная   │
│   │ Running │   │         │  инфраструктура │
│   └────┬────┘   │         │                 │
│        │        │         │   ┌─────────┐   │
│   ┌────▼────┐   │  Sync   │   │Database │   │
│   │Database │───┼────────►│   │ Replica │   │
│   └─────────┘   │         │   └─────────┘   │
└─────────────────┘         └─────────────────┘

При failover:
- Запуск compute ресурсов
- Масштабирование инфраструктуры
- Переключение DNS

Характеристики:
- Минимальные затраты в обычное время
- RTO: часы (время запуска инфраструктуры)
- Стоимость: ~1.2x от single-region
```

## Репликация данных

### Синхронная vs Асинхронная

```yaml
synchronous_replication:
  description: "Запись подтверждается после записи во все реплики"
  pros:
    - RPO = 0 (нет потери данных)
    - Строгая консистентность
  cons:
    - Увеличенная latency (зависит от расстояния)
    - Недоступность при сетевых проблемах
  use_cases:
    - Финансовые транзакции
    - Критичные данные

asynchronous_replication:
  description: "Запись подтверждается сразу, реплицируется в фоне"
  pros:
    - Низкая latency
    - Устойчивость к сетевым проблемам
  cons:
    - RPO > 0 (возможна потеря данных)
    - Eventual consistency
  use_cases:
    - Большинство систем
    - Аналитические данные
```

### PostgreSQL Cross-Region

```yaml
# Patroni конфигурация для cross-region
patroni_config:
  scope: myapp-cluster

  postgresql:
    parameters:
      synchronous_commit: "remote_apply"
      synchronous_standby_names: "region_b"

  # Streaming replication
  replication:
    slots:
      region_b_slot:
        type: physical

---
# Terraform: RDS Cross-Region Read Replica
resource "aws_db_instance" "replica" {
  provider               = aws.region_b
  replicate_source_db    = aws_db_instance.primary.arn
  instance_class         = "db.r5.xlarge"

  # Для failover
  backup_retention_period = 7

  tags = {
    Role = "dr-replica"
  }
}
```

### MongoDB Multi-Region

```javascript
// MongoDB replica set конфигурация
rs.initiate({
  _id: "myapp-rs",
  members: [
    // Primary region
    { _id: 0, host: "mongo-0.region-a:27017", priority: 10 },
    { _id: 1, host: "mongo-1.region-a:27017", priority: 9 },
    // DR region
    { _id: 2, host: "mongo-0.region-b:27017", priority: 5 },
    { _id: 3, host: "mongo-1.region-b:27017", priority: 4 },
    // Arbiter
    { _id: 4, host: "mongo-arbiter.region-a:27017", arbiterOnly: true }
  ],
  settings: {
    getLastErrorModes: {
      multiRegion: { "region": 2 }
    }
  }
});

// Write concern для критичных операций
db.orders.insertOne(
  { ... },
  { writeConcern: { w: "multiRegion", wtimeout: 5000 } }
)
```

### Redis Cross-Region

```yaml
# Redis Enterprise Active-Active
clusters:
  - name: region-a-cluster
    region: us-east-1
    nodes: 3

  - name: region-b-cluster
    region: eu-west-1
    nodes: 3

crdb_config:
  name: myapp-crdb
  memory_size: 10GB
  replication: true
  causal_consistency: true

  # Conflict resolution
  conflict_resolution:
    # Last Write Wins
    default: lww
    # Custom для counters
    counters: add-only
```

## DNS и маршрутизация

### Route 53 Health Checks

```yaml
# Terraform: Health check и failover
resource "aws_route53_health_check" "primary" {
  fqdn              = "api-primary.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10

  regions = ["us-east-1", "eu-west-1", "ap-southeast-1"]

  tags = {
    Name = "primary-health-check"
  }
}

resource "aws_route53_record" "api" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"

  # Failover routing
  failover_routing_policy {
    type = "PRIMARY"
  }

  set_identifier  = "primary"
  health_check_id = aws_route53_health_check.primary.id

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}
```

### Global Load Balancer

```yaml
# GCP Global Load Balancer
resource "google_compute_global_forwarding_rule" "default" {
  name       = "global-rule"
  target     = google_compute_target_https_proxy.default.id
  port_range = "443"
  ip_address = google_compute_global_address.default.address
}

resource "google_compute_backend_service" "default" {
  name                  = "backend-service"
  load_balancing_scheme = "EXTERNAL"

  backend {
    group           = google_compute_region_instance_group_manager.region_a.instance_group
    balancing_mode  = "UTILIZATION"
    capacity_scaler = 1.0
  }

  backend {
    group           = google_compute_region_instance_group_manager.region_b.instance_group
    balancing_mode  = "UTILIZATION"
    capacity_scaler = 1.0
  }

  health_checks = [google_compute_health_check.default.id]
}
```

## Kubernetes Multi-Cluster

### Federation с Kubefed

```yaml
# Federated Deployment
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: myapp
      template:
        spec:
          containers:
            - name: myapp
              image: myapp:1.0
  placement:
    clusters:
      - name: cluster-region-a
      - name: cluster-region-b
  overrides:
    - clusterName: cluster-region-a
      clusterOverrides:
        - path: "/spec/replicas"
          value: 5
    - clusterName: cluster-region-b
      clusterOverrides:
        - path: "/spec/replicas"
          value: 3
```

### Submariner для cross-cluster networking

```bash
# Установка Submariner
subctl deploy-broker --kubeconfig ~/.kube/config

# Присоединение кластеров
subctl join broker-info.subm --clusterid cluster-a --natt=false
subctl join broker-info.subm --clusterid cluster-b --natt=false

# Экспорт сервиса
kubectl annotate service myapp-svc \
  submariner.io/globalnet-egressIP=cluster-a
```

### Istio Multi-Cluster

```yaml
# Istio multi-cluster конфигурация
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-multicluster
spec:
  profile: default
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster-a
      network: network-a
  meshConfig:
    defaultConfig:
      proxyMetadata:
        ISTIO_META_DNS_CAPTURE: "true"
        ISTIO_META_DNS_AUTO_ALLOCATE: "true"
```

## Процедуры Failover

### Автоматический Failover

```python
# Автоматический failover контроллер
class FailoverController:
    def __init__(self, config):
        self.primary_region = config['primary']
        self.dr_region = config['dr']
        self.health_threshold = config['health_threshold']
        self.failover_triggered = False

    async def monitor_health(self):
        while True:
            primary_health = await self.check_region_health(self.primary_region)

            if primary_health < self.health_threshold:
                if not self.failover_triggered:
                    await self.initiate_failover()
            elif self.failover_triggered:
                await self.consider_failback()

            await asyncio.sleep(10)

    async def initiate_failover(self):
        """Автоматический failover"""
        logger.critical("Initiating automatic failover")

        # 1. Verify DR region health
        dr_health = await self.check_region_health(self.dr_region)
        if dr_health < 0.9:
            logger.error("DR region not healthy enough for failover")
            await self.alert_oncall("Manual intervention required")
            return

        # 2. Stop writes to primary (if possible)
        await self.fence_primary_region()

        # 3. Wait for replication lag
        await self.wait_for_replication_sync(timeout=60)

        # 4. Promote DR database
        await self.promote_dr_database()

        # 5. Update DNS
        await self.update_dns_to_dr()

        # 6. Scale up DR compute
        await self.scale_dr_region()

        self.failover_triggered = True
        await self.notify_stakeholders("Failover completed")
```

### Runbook для ручного Failover

```yaml
# Runbook: Manual Regional Failover
runbook:
  name: "Regional Failover"
  trigger: "Primary region unavailable > 5 minutes"

  steps:
    - name: "Assess Situation"
      actions:
        - "Confirm primary region outage via multiple sources"
        - "Check DR region health"
        - "Notify incident commander"
      duration: "5 minutes"

    - name: "Decision Point"
      actions:
        - "Determine if failover is necessary"
        - "Get approval from incident commander"
        - "Document decision rationale"

    - name: "Pre-Failover Checks"
      actions:
        - "Verify DR database replication lag"
        - "Confirm DR compute capacity"
        - "Test DR region connectivity"
      commands:
        - "kubectl --context=dr-cluster get nodes"
        - "psql -h dr-db -c 'SELECT pg_last_wal_replay_lsn()'"

    - name: "Execute Failover"
      actions:
        - "Fence primary database"
        - "Promote DR database to primary"
        - "Update DNS records"
        - "Scale DR compute resources"
      commands:
        - "aws rds promote-read-replica --db-instance-identifier dr-replica"
        - "aws route53 change-resource-record-sets --hosted-zone-id XXX --change-batch file://failover-dns.json"

    - name: "Validation"
      actions:
        - "Verify application health in DR"
        - "Confirm user traffic flowing to DR"
        - "Monitor error rates"
      duration: "15 minutes"

    - name: "Communication"
      actions:
        - "Update status page"
        - "Notify customers"
        - "Internal communication"
```

## Тестирование DR

### DR Testing Framework

```yaml
# DR тест план
dr_test_plan:
  frequency: "quarterly"
  duration: "4 hours"

  scenarios:
    - name: "Database Failover"
      description: "Тест failover базы данных"
      steps:
        - "Promote read replica"
        - "Verify application connectivity"
        - "Test write operations"
      success_criteria:
        - "RTO < 15 minutes"
        - "No data loss (RPO = 0)"
        - "All applications functional"

    - name: "Full Region Failover"
      description: "Полный failover региона"
      steps:
        - "Simulate region outage"
        - "Execute failover procedure"
        - "Verify all services in DR"
      success_criteria:
        - "RTO < 30 minutes"
        - "RPO < 5 minutes"
        - "99% functionality available"

    - name: "Failback Test"
      description: "Возврат в primary регион"
      steps:
        - "Restore primary region"
        - "Sync data from DR"
        - "Execute failback"
      success_criteria:
        - "Clean failback"
        - "No data inconsistency"
```

### Chaos Engineering для DR

```yaml
# Chaos Mesh: симуляция регионального сбоя
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: region-isolation
spec:
  action: partition
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      region: "primary"
  direction: both
  target:
    selector:
      namespaces:
        - production
      labelSelectors:
        region: "secondary"
  duration: "30m"
```

## Мониторинг Multi-Region

### Метрики

```yaml
# Prometheus правила для multi-region
groups:
  - name: multi-region-alerts
    rules:
      - alert: ReplicationLagHigh
        expr: |
          pg_replication_lag_seconds > 60
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Database replication lag > 60s"

      - alert: RegionHealthDegraded
        expr: |
          region_health_score < 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Region health degraded"

      - alert: CrossRegionLatencyHigh
        expr: |
          histogram_quantile(0.99,
            rate(cross_region_request_duration_seconds_bucket[5m])
          ) > 0.5
        for: 10m
        labels:
          severity: warning
```

### Dashboard

```yaml
# Grafana dashboard для multi-region
dashboard:
  title: "Multi-Region Overview"

  panels:
    - title: "Region Health"
      type: "stat"
      targets:
        - expr: "region_health_score"
          legendFormat: "{{region}}"

    - title: "Replication Lag"
      type: "timeseries"
      targets:
        - expr: "pg_replication_lag_seconds"

    - title: "Traffic Distribution"
      type: "piechart"
      targets:
        - expr: "sum(rate(http_requests_total[5m])) by (region)"

    - title: "Cross-Region Latency"
      type: "heatmap"
      targets:
        - expr: "cross_region_latency_seconds_bucket"
```

## Стоимость Multi-Region

### Компоненты затрат

```yaml
cost_components:
  compute:
    description: "Серверы и контейнеры"
    active_active: "2x"
    active_passive: "1.3-1.5x"
    pilot_light: "1.1x"

  storage:
    description: "Диски и хранилище"
    multiplier: "2x для всех паттернов"

  data_transfer:
    description: "Межрегиональный трафик"
    note: "Может быть значительным"
    optimization:
      - "Compression"
      - "Batching"
      - "Smart replication"

  database:
    description: "Managed database реплики"
    cross_region_replica: "+30-50%"

  networking:
    description: "VPN, Direct Connect, Transit Gateway"
    fixed_cost: "Существенный"
```

### ROI расчёт

```python
# ROI калькулятор для DR
def calculate_dr_roi(
    annual_revenue: float,
    hourly_downtime_cost: float,
    expected_outages_per_year: int,
    avg_outage_duration_hours: float,
    dr_annual_cost: float,
    dr_rto_hours: float
) -> dict:
    """
    Расчёт ROI для DR решения.
    """
    # Стоимость простоя без DR
    cost_without_dr = (
        expected_outages_per_year *
        avg_outage_duration_hours *
        hourly_downtime_cost
    )

    # Стоимость простоя с DR
    cost_with_dr = (
        expected_outages_per_year *
        dr_rto_hours *
        hourly_downtime_cost
    )

    # Сэкономлено
    savings = cost_without_dr - cost_with_dr

    # ROI
    roi = (savings - dr_annual_cost) / dr_annual_cost * 100

    return {
        "cost_without_dr": cost_without_dr,
        "cost_with_dr": cost_with_dr,
        "annual_savings": savings,
        "dr_investment": dr_annual_cost,
        "roi_percent": roi,
        "payback_months": dr_annual_cost / (savings / 12) if savings > 0 else float('inf')
    }

# Пример
result = calculate_dr_roi(
    annual_revenue=10_000_000,
    hourly_downtime_cost=50_000,
    expected_outages_per_year=2,
    avg_outage_duration_hours=4,
    dr_annual_cost=200_000,
    dr_rto_hours=0.5
)
# ROI: 75%, Payback: 6 months
```

## Best Practices

### 1. Начинайте с данных

```
Приоритеты:
1. Репликация базы данных
2. Репликация объектного хранилища
3. Репликация конфигурации
4. Синхронизация секретов
5. Compute инфраструктура
```

### 2. Регулярное тестирование

```yaml
testing_schedule:
  weekly:
    - "Health check verification"
    - "Replication lag monitoring"

  monthly:
    - "Database failover drill"
    - "DNS failover test"

  quarterly:
    - "Full region failover"
    - "Failback procedure"
    - "Runbook review"

  annually:
    - "Extended outage simulation"
    - "Third-party DR audit"
```

### 3. Документация

```yaml
required_documentation:
  - name: "Architecture diagram"
    update_frequency: "On change"

  - name: "Failover runbooks"
    update_frequency: "Quarterly"

  - name: "Contact list"
    update_frequency: "Monthly"

  - name: "RTO/RPO requirements"
    update_frequency: "Annually"

  - name: "DR test reports"
    retention: "3 years"
```

## Anti-patterns

### 1. DR как afterthought
```
❌ "Сделаем DR потом"
✅ DR проектируется вместе с основной архитектурой
```

### 2. Непротестированный DR
```
❌ "У нас есть DR, но мы его не тестировали"
✅ Регулярные DR drills с измерением RTO/RPO
```

### 3. Игнорирование данных
```
❌ Compute в DR без репликации данных
✅ Data-first подход к DR
```

### 4. Ручной failover без автоматизации
```
❌ "Мы вручную переключим DNS"
✅ Автоматизированный failover с ручным override
```

### 5. Одинаковый DR для всех сервисов
```
❌ "Всё в Active-Active"
✅ Tier-based подход к DR
```

## Checklist

### Планирование
- [ ] Определены RTO/RPO для каждого сервиса
- [ ] Классифицированы сервисы по tier'ам
- [ ] Выбран паттерн DR для каждого tier'а
- [ ] Рассчитана стоимость и ROI

### Реализация
- [ ] Настроена репликация данных
- [ ] Развёрнута инфраструктура в DR регионе
- [ ] Настроен DNS failover
- [ ] Автоматизированы процедуры failover

### Тестирование
- [ ] Проведён первичный DR drill
- [ ] Измерены реальные RTO/RPO
- [ ] Создан график регулярных тестов
- [ ] Документированы результаты тестов

### Операции
- [ ] Созданы runbooks для всех сценариев
- [ ] Настроен мониторинг репликации
- [ ] Определена on-call процедура
- [ ] Обучена команда

## Ссылки

- [AWS Disaster Recovery](https://aws.amazon.com/disaster-recovery/)
- [GCP DR Planning Guide](https://cloud.google.com/architecture/dr-scenarios-planning-guide)
- [Azure Business Continuity](https://docs.microsoft.com/en-us/azure/availability-zones/az-overview)
- [PostgreSQL Replication](https://www.postgresql.org/docs/current/high-availability.html)
- [Kubernetes Federation](https://github.com/kubernetes-sigs/kubefed)
