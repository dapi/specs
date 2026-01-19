# SRO-002 Chaos Engineering

Chaos Engineering — a discipline of experimenting on distributed systems to uncover weaknesses and improve resilience to failures.

## Why Chaos Engineering?

### Proactive Problem Discovery
- Find vulnerabilities **before** they become incidents
- Verify assumptions about system behavior
- Identify hidden dependencies

### Improved Resilience
- Train teams on real failure scenarios
- Verify recovery mechanisms
- Improve runbooks and documentation

### System Confidence
- Prove fault tolerance works
- Confidence in scaling
- Readiness for peak loads

## Principles of Chaos Engineering

### 1. Start with a Hypothesis
```
Hypothesis: "When Redis fails, the service will correctly switch
to fallback and continue serving requests"

Success Metrics:
- Error rate < 1%
- Latency P99 < 500ms
- Availability > 99.9%
```

### 2. Minimize Blast Radius
```
Impact Levels:
1. Development → Staging → Production
2. 1 pod → 1 node → 1 AZ → 1 region
3. 1% traffic → 10% → 100%
```

### 3. Run in Production
> "The only way to know how a system behaves in production
> is to run experiments in production"
> — Netflix

### 4. Automate
```yaml
# Regular experiments
schedule:
  pod_failure:
    cron: "0 10 * * 1-5"  # Every weekday at 10:00
    duration: 5m
  network_latency:
    cron: "0 14 * * 3"    # Every Wednesday at 14:00
    duration: 15m
```

## Types of Experiments

### 1. Infrastructure Failures

#### Pod/Container Termination
```yaml
# Chaos Mesh: kill random pod
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
# Chaos Mesh: node shutdown
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
# Adding 100ms delay
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
# 10% packet loss
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
# Isolate service from database
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
# Load CPU to 80%
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
# Allocate 512MB memory
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
# Simulate slow queries
class DatabaseChaos:
    def inject_slow_queries(self, percentage: int, delay_ms: int):
        """Add delay to percentage of queries."""
        pass

    def inject_connection_failures(self, percentage: int):
        """Simulate connection failures."""
        pass

    def inject_deadlocks(self):
        """Create artificial deadlocks."""
        pass
```

## Tools

### Platform Comparison

| Tool | Kubernetes | AWS | Type | Features |
|------|------------|-----|------|----------|
| Chaos Mesh | Yes | No | Open Source | CNCF, powerful UI |
| Litmus | Yes | Yes | Open Source | ChaosHub with ready experiments |
| Gremlin | Yes | Yes | SaaS | Enterprise features |
| AWS FIS | No | Yes | SaaS | Native AWS integration |
| Chaos Monkey | Yes | Yes | Open Source | Netflix, classic |

### Installing Chaos Mesh

```bash
# Install via Helm
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-mesh \
  --create-namespace \
  --set dashboard.securityMode=false
```

### Installing Litmus

```bash
# Install LitmusChaos
kubectl apply -f https://litmuschaos.github.io/litmus/litmus-operator-v2.14.0.yaml

# Install ChaosCenter
helm install chaos litmuschaos/litmus \
  --namespace litmus \
  --create-namespace
```

## Game Days

### What is a Game Day?
A planned event where the team runs chaos experiments and practices incident response.

### Game Day Structure

```
09:00 - Introduction and scenario overview
09:30 - Verify monitoring and alerts
10:00 - Experiment #1: Pod failure
10:30 - Results analysis
11:00 - Experiment #2: Network latency
11:30 - Results analysis
12:00 - Lunch
13:00 - Experiment #3: Database failover
13:30 - Results analysis
14:00 - Experiment #4: Full AZ failure
14:30 - Results analysis
15:00 - Retrospective and action items
16:00 - Wrap-up
```

### Preparation Checklist

```markdown
## Before Game Day
- [ ] Experiment scenarios defined
- [ ] Chaos engineering tools configured
- [ ] Rollback procedures verified
- [ ] Stakeholders notified
- [ ] Runbooks prepared
- [ ] Monitoring and dashboards configured

## During Game Day
- [ ] All participants in Slack/Teams channel
- [ ] Monitoring visible on separate screen
- [ ] Timeline of events recorded
- [ ] Unexpected results documented

## After Game Day
- [ ] Report with findings written
- [ ] Tickets created for fixes
- [ ] Runbooks updated
- [ ] Next Game Day scheduled
```

### Report Template

```markdown
# Game Day Report: 2024-01-15

## Participants
- SRE: @alice, @bob
- Dev: @charlie, @diana

## Experiments

### Experiment 1: Pod Failure
- **Hypothesis**: System survives loss of 1 pod without degradation
- **Result**: PASSED ✅
- **Observations**:
  - Failover took 8 seconds
  - No client errors recorded

### Experiment 2: Network Latency 200ms
- **Hypothesis**: P99 latency stays < 1s
- **Result**: FAILED ❌
- **Observations**:
  - P99 reached 2.5s
  - Timeouts in payment service
- **Action Items**:
  - [ ] PROJ-123: Increase timeout in payment client
  - [ ] PROJ-124: Add circuit breaker

## Summary
- Passed: 3/4
- Failed: 1/4
- Action Items: 5
```

## Metrics and Monitoring

### Key Metrics During Experiments

```yaml
# Prometheus queries for monitoring
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

### Dashboard for Chaos Experiments

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

### 1. Start Small
```
Week 1: Experiments in dev environment
Week 2: Experiments in staging
Week 3: Experiments in production (1% traffic)
Month 2: Regular experiments in production
```

### 2. Always Have a "STOP" Button
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

### 3. Document Everything
```yaml
experiment:
  name: "database-failover"
  description: "Testing PostgreSQL failover"
  hypothesis: "System will switch to replica in <30s"
  rollback: "kubectl delete podchaos database-failover"
  owner: "team-platform"
  approved_by: "alice@company.com"
  last_run: "2024-01-15T10:00:00Z"
  results_url: "https://wiki/chaos/db-failover-2024-01"
```

### 4. Integrate with CI/CD
```yaml
# GitHub Actions: chaos tests before release
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

## Anti-patterns

### 1. Chaos Without Monitoring
```
❌ Run experiment and see "what breaks"
✅ Define success metrics before running experiment
```

### 2. Chaos in Production Without Preparation
```
❌ "Let's just kill a pod in production"
✅ First staging, then limited production
```

### 3. Ignoring Results
```
❌ "Test failed, but no time to fix now"
✅ Create ticket, prioritize, fix
```

## References

- [Principles of Chaos Engineering](https://principlesofchaos.org/)
- [Netflix Chaos Engineering](https://netflixtechblog.com/tagged/chaos-engineering)
- [Chaos Mesh Documentation](https://chaos-mesh.org/docs/)
- [Gremlin Chaos Engineering Guide](https://www.gremlin.com/community/tutorials/chaos-engineering-the-history-principles-and-practice/)
- [Google DiRT (Disaster Recovery Testing)](https://sre.google/sre-book/accelerating-sre-on-call/)
