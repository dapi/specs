# SRP-006 Auto-scaling

## Definition

Auto-scaling is the automatic increase or decrease of the number of service instances depending on current load and resource consumption.

## Types of auto-scaling

### 1. Horizontal Pod Autoscaler (HPA)

Increase/decrease the number of pods.

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

Increase/decrease resources (CPU, Memory) for each pod.

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
    updateMode: "Auto"  # or "Off", "Initial", "Recreate"
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

Add/remove nodes in the cluster.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler
  namespace: kube-system
data:
  # Minimum and maximum number of nodes
  minNodes: "3"
  maxNodes: "50"

  # Scale when unable to schedule pod
  expander: least-waste

  # Remove nodes after 10 minutes of idleness
  scale-down-unneeded-time: 10m

  # Don't remove nodes with pods from kube-system
  skip-nodes-with-system-pods: true

  # Scaling boundaries by zones
  balancing-ignore-label: topology.kubernetes.io/zone
```

## Scaling metrics

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

Needed for more accurate scaling:

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

## Scaling thresholds

### Scaling speed

```yaml
behavior:
  scaleUp:
    # Don't scale if we just scaled down
    stabilizationWindowSeconds: 60

    # Scaling rules
    policies:
    # Can double count in 15 seconds
    - type: Percent
      value: 100
      periodSeconds: 15

    # Or add maximum 2 pods in 60 seconds
    - type: Pods
      value: 2
      periodSeconds: 60

    # Choose the most conservative (larger)
    selectPolicy: Max

  scaleDown:
    # Wait 5 minutes before decreasing
    stabilizationWindowSeconds: 300

    # Reduce by no more than 10% in 60 seconds
    policies:
    - type: Percent
      value: 10
      periodSeconds: 60

    # Choose the most aggressive (smaller)
    selectPolicy: Min
```

### Strategies

**Conservative (conservative):**
- Scale up slowly
- Scale down very slowly
- Suitable for stable services

```yaml
scaleUp:
  stabilizationWindowSeconds: 300
  policies:
  - type: Pods
    value: 1
    periodSeconds: 60
```

**Aggressive (aggressive):**
- Scale up quickly
- Scale down quickly
- Suitable for spike load

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

## Configuration via environment variables

```
# Basic settings
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

## Graceful Shutdown when scaling down

When decreasing the number of replicas, it's important to properly terminate work:

```python
import signal
import sys

shutdown_in_progress = False

def handle_sigterm(signum, frame):
    global shutdown_in_progress
    shutdown_in_progress = True

    # 1. Stop accepting new requests
    stop_accepting_requests()

    # 2. Wait for current requests to complete (30 seconds)
    wait_for_requests(timeout=30)

    # 3. Stop background jobs
    stop_background_jobs()

    # 4. Close connections
    close_database_connections()
    close_redis_connections()

    # 5. Exit
    sys.exit(0)

signal.signal(signal.SIGTERM, handle_sigterm)
```

## Monitoring

### Metrics

```
kube_horizontalpodautoscaler_spec_max_replicas (gauge)
kube_horizontalpodautoscaler_spec_min_replicas (gauge)
kube_horizontalpodautoscaler_status_current_replicas (gauge)
kube_horizontalpodautoscaler_status_desired_replicas (gauge)

# Detailed scaling information
kube_horizontalpodautoscaler_status_condition (gauge)
  condition=AbleToScale
  condition=ScalingActive
  condition=ScalingLimited
```

### Alerting

Alerts should fire when:

```yaml
# HPA at maximum for extended time
- alert: HPAAtMax
  expr: |
    kube_horizontalpodautoscaler_status_current_replicas
    ==
    kube_horizontalpodautoscaler_spec_max_replicas
  for: 15m
  annotations:
    message: "HPA is at max replicas for more than 15 minutes"

# Unable to scale
- alert: HPAScalingLimited
  expr: |
    kube_horizontalpodautoscaler_status_condition{condition="ScalingLimited",status="true"} == 1
  for: 5m
  annotations:
    message: "HPA is unable to scale"

# CPU above target
- alert: HighCPUUsage
  expr: |
    avg(container_cpu_usage_seconds_total{container!=""})
    > 0.8
  for: 10m
  annotations:
    message: "High CPU usage detected"
```

## Testing auto-scaling

### Load Testing

```bash
# Create load and check scaling
k6 run --vus 100 --duration 5m script.js

# Check number of pods
kubectl get pods -l app=my-app -w
watch -n 1 kubectl get hpa
```

### Chaos Engineering

```bash
# Simulate pod failures
kubectl delete pods -l app=my-app --grace-period=0 --force

# Check that new pods are created
kubectl get pods -l app=my-app -w
```

## Limits and constraints

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
  minAvailable: 2  # Minimum 2 pods must be available
  # maxUnavailable: 1  # Or maximum 1 pod can be unavailable
  selector:
    matchLabels:
      app: my-app
```

## Best practices

✅ **Do**
* Configure minReplicas for fault tolerance
* Limit maxReplicas by resources/budget
* Set readiness probes for correct scaling
* Use multiple metrics (CPU + custom)
* Test under load
* Set PDBs

❌ **Don't**
* Unlimited maxReplicas
* Leave default settings without analysis
* Scale only by CPU
* Ignore resource requests/limits
* Disable stabilization windows

## Additional resources

* [Kubernetes HPA Documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
* [Kubernetes VPA Documentation](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
* [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
* [KEDA - Kubernetes Event-driven Autoscaling](https://keda.sh/)
* [Custom Metrics Adapter](https://github.com/kubernetes-sigs/prometheus-adapter)
