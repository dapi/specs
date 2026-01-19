# SRP-006 Auto-scaling (Автоматическое масштабирование)

## Определение

Auto-scaling - это автоматическое увеличение или уменьшение количества экземпляров сервиса в зависимости от текущей нагрузки и потребления ресурсов.

## Типы автомасштабирования

### 1. Horizontal Pod Autoscaler (HPA)

Увеличение/уменьшение количества pod-ов.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 2
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

### 2. Vertical Pod Autoscaler (VPA)

Увеличение/уменьшение ресурсов (CPU, Memory) для каждого pod-а.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app-deployment
  updatePolicy:
    updateMode: "Auto"  # или "Off", "Initial", "Recreate"
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: 100m
        memory: 256Mi
      maxAllowed:
        cpu: 1
        memory: 2Gi
      controlledResources: ["cpu", "memory"]
```

### 3. Cluster Autoscaler

Добавление/удаление узлов в кластере.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler
  namespace: kube-system
data:
  # Минимальное и максимальное количество узлов
  minNodes: "3"
  maxNodes: "50"

  # Масштабировать при невозможности запланировать pod
  expander: least-waste

  # Удалять узлы после 10 минут простоя
  scale-down-unneeded-time: 10m

  # Не удалять узлы с pods из kube-system
  skip-nodes-with-system-pods: true

  # Границы масштабирования по зонам
  balancing-ignore-label: topology.kubernetes.io/zone
```

## Метрики для масштабирования

### CPU/Memory
```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 70
```

### Custom Metrics

Нужны для более точного масштабирования:

```yaml
# Prometheus Adapter
- type: Pods
  pods:
    metric:
      name: http_requests_per_second
    target:
      type: AverageValue
      averageValue: "1000"

# Queue length
- type: Object
  object:
    metric:
      name: queue_length
    describedObject:
      apiVersion: messaging.k8s.io/v1
      kind: Queue
      name: worker-queue
    target:
      type: Value
      value: "100"
```

### Business Metrics

```yaml
# Latency
- type: Pods
  pods:
    metric:
      name: http_request_duration_seconds
    target:
      type: AverageValue
      averageValue: "0.1"  # 100ms

# Error Rate
- type: Pods
  pods:
    metric:
      name: http_requests_errors_total
    target:
      type: AverageValue
      averageValue: "10"  # 10 errors per second max
```

## Пороги масштабирования

### Скорость масштабирования

```yaml
behavior:
  scaleUp:
    # Не масштабировать, если только что масштабировали вниз
    stabilizationWindowSeconds: 60

    # Правила масштабирования
    policies:
    # Можно удвоить количество за 15 секунд
    - type: Percent
      value: 100
      periodSeconds: 15

    # Или добавить максимум 2 pod-а за 60 секунд
    - type: Pods
      value: 2
      periodSeconds: 60

    # Выбирается наиболее консервативный (больший)
    selectPolicy: Max

  scaleDown:
    # Ждать 5 минут перед уменьшением
    stabilizationWindowSeconds: 300

    # Уменьшать не более чем на 10% за 60 секунд
    policies:
    - type: Percent
      value: 10
      periodSeconds: 60

    # Выбирается наиболее агрессивный (меньший)
    selectPolicy: Min
```

### Стратегии

**Conservative (консервативная):**
- Медленно масштабировать вверх
- Очень медленно масштабировать вниз
- Подходит для стабильных сервисов

```yaml
scaleUp:
  stabilizationWindowSeconds: 300
  policies:
  - type: Pods
    value: 1
    periodSeconds: 60
```

**Aggressive (агрессивная):**
- Быстро масштабировать вверх
- Быстро масштабировать вниз
- Подходит для спиковой нагрузки

```yaml
scaleUp:
  stabilizationWindowSeconds: 30
  policies:
  - type: Percent
    value: 100
    periodSeconds: 15

scaleDown:
  stabilizationWindowSeconds: 60
  policies:
  - type: Percent
    value: 50
    periodSeconds: 60
```

## Конфигурация через переменные окружения

```
# Основные настройки
AUTOSCALING_ENABLED=true
AUTOSCALING_MIN_REPLICAS=2
AUTOSCALING_MAX_REPLICAS=20

# CPU thresholds
AUTOSCALING_CPU_TARGET=70
AUTOSCALING_CPU_ENABLED=true

# Memory thresholds
AUTOSCALING_MEMORY_TARGET=80
AUTOSCALING_MEMORY_ENABLED=true

# Custom metrics
AUTOSCALING_REQUESTS_PER_SECOND_TARGET=1000
AUTOSCALING_CUSTOM_METRICS_ENABLED=false

# Scaling speed
AUTOSCALING_SCALE_UP_STABILIZATION=60
AUTOSCALING_SCALE_DOWN_STABILIZATION=300
AUTOSCALING_MAX_SCALEUP_PODS=2
AUTOSCALING_MAX_SCALEUP_INTERVAL=60
```

## Graceful Shutdown при масштабировании вниз

При уменьшении количества реплик важно корректно завершить работу:

```python
import signal
import sys

shutdown_in_progress = False

def handle_sigterm(signum, frame):
    global shutdown_in_progress
    shutdown_in_progress = True

    # 1. Остановить принятие новых запросов
    stop_accepting_requests()

    # 2. Дождаться завершения текущих запросов (30 секунд)
    wait_for_requests(timeout=30)

    # 3. Завершить фоновые задачи
    stop_background_jobs()

    # 4. Закрыть соединения
    close_database_connections()
    close_redis_connections()

    # 5. Выход
    sys.exit(0)

signal.signal(signal.SIGTERM, handle_sigterm)
```

## Мониторинг

### Метрики

```
kube_horizontalpodautoscaler_spec_max_replicas (gauge)
kube_horizontalpodautoscaler_spec_min_replicas (gauge)
kube_horizontalpodautoscaler_status_current_replicas (gauge)
kube_horizontalpodautoscaler_status_desired_replicas (gauge)

# Подробная информация о масштабировании
kube_horizontalpodautoscaler_status_condition (gauge)
  condition=AbleToScale
  condition=ScalingActive
  condition=ScalingLimited
```

### Alerting

Алерты должны срабатывать при:

```yaml
# ХПА на максимуме длительное время
- alert: HPAAtMax
  expr: |
    kube_horizontalpodautoscaler_status_current_replicas
    ==
    kube_horizontalpodautoscaler_spec_max_replicas
  for: 15m
  annotations:
    message: "HPA is at max replicas for more than 15 minutes"

# Невозможность масштабировать
- alert: HPAScalingLimited
  expr: |
    kube_horizontalpodautoscaler_status_condition{condition="ScalingLimited",status="true"} == 1
  for: 5m
  annotations:
    message: "HPA is unable to scale"

# ЦПУ выше целевого значения
- alert: HighCPUUsage
  expr: |
    avg(container_cpu_usage_seconds_total{container!=""})
    > 0.8
  for: 10m
  annotations:
    message: "High CPU usage detected"
```

## Тестирование автомасштабирования

### Load Testing

```bash
# Создаем нагрузку и проверяем масштабирование
k6 run --vus 100 --duration 5m script.js

# Проверяем количество pod-ов
kubectl get pods -l app=my-app -w
watch -n 1 kubectl get hpa
```

### Chaos Engineering

```bash
# Симулируем отказ pod-ов
kubectl delete pods -l app=my-app --grace-period=0 --force

# Проверяем, что новые pod-ы создаются
kubectl get pods -l app=my-app -w
```

## Лимиты и ограничения

### Namespace Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: app-quota
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "20"
```

### Pod Disruption Budgets

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2  # Минимум 2 pod-а должны быть доступны
  # maxUnavailable: 1  # Или максимум 1 pod может быть недоступен
  selector:
    matchLabels:
      app: my-app
```

## Best practices

✅ **Делать**
* Настраивать minReplicas для отказоустойчивости
* Ограничивать maxReplicas по ресурсам/бюджету
* Устанавливать readiness probes для корректного масштабирования
* Использовать multiple metrics (CPU + custom)
* Тестировать при нагрузке
* Устанавливать PDBs

❌ **Не делать**
* maxReplicas без ограничений
* Оставлять default settings без анализа
* Масштабировать только по CPU
* Игнорировать resource requests/limits
* Отключать stabilization windows

## Дополнительные ресурсы

* [Kubernetes HPA Documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
* [Kubernetes VPA Documentation](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
* [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
* [KEDA - Kubernetes Event-driven Autoscaling](https://keda.sh/)
* [Custom Metrics Adapter](https://github.com/kubernetes-sigs/prometheus-adapter)
