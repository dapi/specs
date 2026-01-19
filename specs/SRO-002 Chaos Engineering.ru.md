# SRO-002 Chaos Engineering

Chaos Engineering — дисциплина проведения экспериментов над распределённой системой для выявления слабых мест и повышения устойчивости к сбоям.

## Зачем нужен Chaos Engineering?

### Проактивное обнаружение проблем
- Находим уязвимости **до** того, как они станут инцидентами
- Проверяем предположения о поведении системы
- Выявляем скрытые зависимости

### Повышение устойчивости
- Тренировка команды на реальных сценариях сбоев
- Верификация механизмов восстановления
- Улучшение runbook'ов и документации

### Доверие к системе
- Доказательство работоспособности fault tolerance
- Уверенность в масштабировании
- Готовность к peak нагрузкам

## Принципы Chaos Engineering

### 1. Начните с гипотезы
```
Гипотеза: "При отказе Redis сервис корректно переключится
на fallback и продолжит обслуживать запросы"

Метрики успеха:
- Error rate < 1%
- Latency P99 < 500ms
- Availability > 99.9%
```

### 2. Минимизируйте blast radius
```
Уровни воздействия:
1. Development → Staging → Production
2. 1 pod → 1 node → 1 AZ → 1 region
3. 1% трафика → 10% → 100%
```

### 3. Запускайте в production
> "Единственный способ узнать, как система ведёт себя в production —
> это проводить эксперименты в production"
> — Netflix

### 4. Автоматизируйте
```yaml
# Регулярные эксперименты
schedule:
  pod_failure:
    cron: "0 10 * * 1-5"  # Каждый будний день в 10:00
    duration: 5m
  network_latency:
    cron: "0 14 * * 3"    # Каждую среду в 14:00
    duration: 15m
```

## Типы экспериментов

### 1. Infrastructure Failures

#### Pod/Container Termination
```yaml
# Chaos Mesh: убийство случайного пода
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-experiment
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      app: api-service
  scheduler:
    cron: "@every 1h"
```

#### Node Failure
```yaml
# Chaos Mesh: отключение ноды
apiVersion: chaos-mesh.org/v1alpha1
kind: PhysicalMachineChaos
metadata:
  name: node-failure
spec:
  action: node-stop
  address:
    - 192.168.1.100:31767
  duration: "5m"
```

### 2. Network Chaos

#### Latency Injection
```yaml
# Добавление задержки 100ms
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay
spec:
  action: delay
  mode: all
  selector:
    labelSelectors:
      app: payment-service
  delay:
    latency: "100ms"
    jitter: "20ms"
    correlation: "50"
  duration: "5m"
```

#### Packet Loss
```yaml
# Потеря 10% пакетов
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: packet-loss
spec:
  action: loss
  mode: all
  selector:
    labelSelectors:
      app: api-service
  loss:
    loss: "10"
    correlation: "50"
  duration: "5m"
```

#### Network Partition
```yaml
# Изоляция сервиса от базы данных
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: db-partition
spec:
  action: partition
  mode: all
  selector:
    labelSelectors:
      app: api-service
  direction: both
  target:
    selector:
      labelSelectors:
        app: postgres
```

### 3. Resource Stress

#### CPU Stress
```yaml
# Загрузка CPU до 80%
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: cpu-stress
spec:
  mode: one
  selector:
    labelSelectors:
      app: worker-service
  stressors:
    cpu:
      workers: 4
      load: 80
  duration: "10m"
```

#### Memory Pressure
```yaml
# Выделение 512MB памяти
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: memory-stress
spec:
  mode: one
  selector:
    labelSelectors:
      app: api-service
  stressors:
    memory:
      workers: 2
      size: "512MB"
  duration: "5m"
```

### 4. Application-Level Chaos

#### HTTP Fault Injection (Istio)
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: api-fault-injection
spec:
  hosts:
    - api-service
  http:
    - fault:
        delay:
          percentage:
            value: 10
          fixedDelay: 5s
        abort:
          percentage:
            value: 5
          httpStatus: 500
      route:
        - destination:
            host: api-service
```

#### Database Chaos
```python
# Симуляция медленных запросов
class DatabaseChaos:
    def inject_slow_queries(self, percentage: int, delay_ms: int):
        """Добавляет задержку к проценту запросов."""
        pass

    def inject_connection_failures(self, percentage: int):
        """Симулирует отказы соединений."""
        pass

    def inject_deadlocks(self):
        """Создаёт искусственные deadlock'и."""
        pass
```

## Инструменты

### Сравнение платформ

| Инструмент | Kubernetes | AWS | Тип | Особенности |
|------------|------------|-----|-----|-------------|
| Chaos Mesh | Да | Нет | Open Source | CNCF, мощный UI |
| Litmus | Да | Да | Open Source | ChaosHub с готовыми экспериментами |
| Gremlin | Да | Да | SaaS | Enterprise features |
| AWS FIS | Нет | Да | SaaS | Нативная интеграция AWS |
| Chaos Monkey | Да | Да | Open Source | Netflix, классика |

### Установка Chaos Mesh

```bash
# Установка через Helm
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-mesh \
  --create-namespace \
  --set dashboard.securityMode=false
```

### Установка Litmus

```bash
# Установка LitmusChaos
kubectl apply -f https://litmuschaos.github.io/litmus/litmus-operator-v2.14.0.yaml

# Установка ChaosCenter
helm install chaos litmuschaos/litmus \
  --namespace litmus \
  --create-namespace
```

## Game Days

### Что такое Game Day?
Запланированное мероприятие, где команда проводит chaos-эксперименты и практикует реагирование на инциденты.

### Структура Game Day

```
09:00 - Вступление и обзор сценариев
09:30 - Проверка мониторинга и алертов
10:00 - Эксперимент #1: Pod failure
10:30 - Разбор результатов
11:00 - Эксперимент #2: Network latency
11:30 - Разбор результатов
12:00 - Обед
13:00 - Эксперимент #3: Database failover
13:30 - Разбор результатов
14:00 - Эксперимент #4: Full AZ failure
14:30 - Разбор результатов
15:00 - Ретроспектива и action items
16:00 - Завершение
```

### Чеклист подготовки

```markdown
## До Game Day
- [ ] Определены сценарии экспериментов
- [ ] Настроены инструменты chaos engineering
- [ ] Проверены rollback процедуры
- [ ] Уведомлены stakeholders
- [ ] Подготовлены runbooks
- [ ] Настроен мониторинг и дашборды

## Во время Game Day
- [ ] Все участники в Slack/Teams канале
- [ ] Мониторинг открыт на отдельном экране
- [ ] Записывается timeline событий
- [ ] Фиксируются неожиданные результаты

## После Game Day
- [ ] Написан отчёт с findings
- [ ] Созданы tickets на исправления
- [ ] Обновлены runbooks
- [ ] Запланирован следующий Game Day
```

### Шаблон отчёта

```markdown
# Game Day Report: 2024-01-15

## Участники
- SRE: @alice, @bob
- Dev: @charlie, @diana

## Эксперименты

### Эксперимент 1: Pod Failure
- **Гипотеза**: Система переживёт потерю 1 пода без деградации
- **Результат**: PASSED ✅
- **Observations**:
  - Failover занял 8 секунд
  - Ошибок клиентов не зафиксировано

### Эксперимент 2: Network Latency 200ms
- **Гипотеза**: P99 latency останется < 1s
- **Результат**: FAILED ❌
- **Observations**:
  - P99 достиг 2.5s
  - Timeout'ы в payment service
- **Action Items**:
  - [ ] PROJ-123: Увеличить timeout в payment client
  - [ ] PROJ-124: Добавить circuit breaker

## Summary
- Passed: 3/4
- Failed: 1/4
- Action Items: 5
```

## Метрики и мониторинг

### Ключевые метрики во время экспериментов

```yaml
# Prometheus queries для мониторинга
queries:
  error_rate: |
    sum(rate(http_requests_total{status=~"5.."}[1m]))
    / sum(rate(http_requests_total[1m])) * 100

  latency_p99: |
    histogram_quantile(0.99,
      sum(rate(http_request_duration_seconds_bucket[1m])) by (le)
    )

  availability: |
    avg_over_time(up{job="api-service"}[5m]) * 100

  saturation: |
    max(container_memory_usage_bytes / container_spec_memory_limit_bytes) * 100
```

### Дашборд для Chaos Experiments

```json
{
  "title": "Chaos Engineering Dashboard",
  "panels": [
    {
      "title": "Error Rate",
      "type": "graph",
      "alert": {
        "threshold": 1,
        "message": "Error rate exceeded 1%"
      }
    },
    {
      "title": "P99 Latency",
      "type": "graph",
      "alert": {
        "threshold": 500,
        "message": "P99 latency exceeded 500ms"
      }
    },
    {
      "title": "Active Experiments",
      "type": "table",
      "datasource": "chaos-mesh"
    }
  ]
}
```

## Best Practices

### 1. Начинайте с малого
```
Неделя 1: Эксперименты в dev окружении
Неделя 2: Эксперименты в staging
Неделя 3: Эксперименты в production (1% трафика)
Месяц 2: Регулярные эксперименты в production
```

### 2. Всегда имейте кнопку "СТОП"
```python
class ChaosExperiment:
    def __init__(self):
        self.abort_flag = threading.Event()

    def run(self):
        while not self.abort_flag.is_set():
            self.execute_chaos()
            time.sleep(1)

    def abort(self):
        self.abort_flag.set()
        self.rollback()
```

### 3. Документируйте всё
```yaml
experiment:
  name: "database-failover"
  description: "Тестирование failover PostgreSQL"
  hypothesis: "Система переключится на replica за <30s"
  rollback: "kubectl delete podchaos database-failover"
  owner: "team-platform"
  approved_by: "alice@company.com"
  last_run: "2024-01-15T10:00:00Z"
  results_url: "https://wiki/chaos/db-failover-2024-01"
```

### 4. Интегрируйте с CI/CD
```yaml
# GitHub Actions: chaos тесты перед релизом
name: Chaos Tests
on:
  push:
    branches: [main]

jobs:
  chaos-test:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to staging
        run: kubectl apply -f k8s/

      - name: Run chaos experiments
        run: |
          kubectl apply -f chaos/pod-failure.yaml
          sleep 300
          kubectl delete -f chaos/pod-failure.yaml

      - name: Verify SLOs
        run: ./scripts/check-slos.sh
```

## Антипаттерны

### 1. Chaos без мониторинга
```
❌ Запускаем эксперимент и смотрим "что сломается"
✅ Определяем метрики успеха до запуска эксперимента
```

### 2. Chaos в production без подготовки
```
❌ "Давайте просто убьём под в production"
✅ Сначала staging, потом ограниченный production
```

### 3. Игнорирование результатов
```
❌ "Тест провалился, но сейчас нет времени чинить"
✅ Создаём ticket, приоритизируем, чиним
```

## Ссылки

- [Principles of Chaos Engineering](https://principlesofchaos.org/)
- [Netflix Chaos Engineering](https://netflixtechblog.com/tagged/chaos-engineering)
- [Chaos Mesh Documentation](https://chaos-mesh.org/docs/)
- [Gremlin Chaos Engineering Guide](https://www.gremlin.com/community/tutorials/chaos-engineering-the-history-principles-and-practice/)
- [Google DiRT (Disaster Recovery Testing)](https://sre.google/sre-book/accelerating-sre-on-call/)
