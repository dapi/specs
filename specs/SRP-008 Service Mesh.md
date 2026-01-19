# SRP-008 Service Mesh

Service Mesh — an infrastructure layer for managing communication between microservices, providing observability, security, and traffic control.

## Why Service Mesh?

### Problems in Microservices Architecture
- Complexity of managing inter-service communication
- Lack of unified security policies
- Difficulties with tracing and debugging
- Heterogeneous implementation of retry, timeout, circuit breaker

### Service Mesh Benefits
- **Observability**: automatic collection of metrics, traces, logs
- **Security**: mTLS between services out of the box
- **Traffic Control**: canary releases, A/B testing, fault injection
- **Policies**: rate limiting, access control at infrastructure level

## Service Mesh Architecture

### Data Plane vs Control Plane

```
┌─────────────────────────────────────────────────────────────┐
│                      Control Plane                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │   istiod    │  │  Citadel    │  │      Galley         │ │
│  │ (Pilot)     │  │ (Security)  │  │   (Config)          │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ xDS API
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                       Data Plane                            │
│  ┌─────────┐      ┌─────────┐      ┌─────────┐            │
│  │ Service │      │ Service │      │ Service │            │
│  │    A    │◄────►│    B    │◄────►│    C    │            │
│  │ ┌─────┐ │      │ ┌─────┐ │      │ ┌─────┐ │            │
│  │ │Envoy│ │      │ │Envoy│ │      │ │Envoy│ │            │
│  │ │Proxy│ │      │ │Proxy│ │      │ │Proxy│ │            │
│  │ └─────┘ │      │ └─────┘ │      │ └─────┘ │            │
│  └─────────┘      └─────────┘      └─────────┘            │
└─────────────────────────────────────────────────────────────┘
```

### Sidecar Pattern

```yaml
# Pod with sidecar proxy
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  annotations:
    sidecar.istio.io/inject: "true"
spec:
  containers:
    - name: my-app
      image: my-app:1.0
      ports:
        - containerPort: 8080
    # Sidecar injected automatically
    # - name: istio-proxy
    #   image: envoyproxy/envoy
```

## Istio

### Installation

```bash
# Install istioctl
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install Istio with demo profile
istioctl install --set profile=demo -y

# Enable automatic sidecar injection
kubectl label namespace default istio-injection=enabled
```

### Installation Profiles

| Profile | Description | Components |
|---------|-------------|------------|
| minimal | Minimal installation | istiod |
| default | Production recommended | istiod, ingress gateway |
| demo | Demo all features | All components |
| empty | Nothing installed | Empty |

### Traffic Management

#### VirtualService

```yaml
# Traffic routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
    - reviews
  http:
    - match:
        - headers:
            end-user:
              exact: jason
      route:
        - destination:
            host: reviews
            subset: v2
    - route:
        - destination:
            host: reviews
            subset: v1
```

#### DestinationRule

```yaml
# Define service versions
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
    loadBalancer:
      simple: ROUND_ROBIN
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

#### Canary Deployment

```yaml
# 90% to v1, 10% to v2
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-canary
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 90
        - destination:
            host: reviews
            subset: v2
          weight: 10
```

### Resiliency

#### Timeout and Retry

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings-timeout
spec:
  hosts:
    - ratings
  http:
    - route:
        - destination:
            host: ratings
      timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 3s
        retryOn: 5xx,reset,connect-failure
```

#### Circuit Breaker

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-circuit-breaker
spec:
  host: reviews
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 30
```

### Security

#### mTLS

```yaml
# Strict mTLS for namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

#### Authorization Policy

```yaml
# Allow access only from frontend
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend
  namespace: default
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/default/sa/frontend"]
      to:
        - operation:
            methods: ["GET", "POST"]
```

#### JWT Validation

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
spec:
  selector:
    matchLabels:
      app: api-gateway
  jwtRules:
    - issuer: "https://auth.example.com"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"
```

## Linkerd

### Installation

```bash
# Install CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

# Check requirements
linkerd check --pre

# Install control plane
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# Verify installation
linkerd check
```

### Injection Annotations

```yaml
# Enable Linkerd for deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    linkerd.io/inject: enabled
spec:
  template:
    metadata:
      annotations:
        linkerd.io/inject: enabled
```

### Service Profiles

```yaml
# Define routes for metrics
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: webapp.default.svc.cluster.local
  namespace: default
spec:
  routes:
    - name: GET /api/users/{id}
      condition:
        method: GET
        pathRegex: /api/users/[^/]+
      isRetryable: true
    - name: POST /api/users
      condition:
        method: POST
        pathRegex: /api/users
      timeout: 5s
```

### Traffic Split (SMI)

```yaml
# Canary deployment with Linkerd
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: webapp-split
spec:
  service: webapp
  backends:
    - service: webapp-stable
      weight: 900m
    - service: webapp-canary
      weight: 100m
```

## Service Mesh Comparison

| Feature | Istio | Linkerd | Consul Connect |
|---------|-------|---------|----------------|
| Proxy | Envoy | linkerd2-proxy | Envoy |
| Complexity | High | Low | Medium |
| Resources | High | Low | Medium |
| mTLS | Yes | Yes | Yes |
| Traffic Management | Full | Basic | Medium |
| Multi-cluster | Yes | Yes | Yes |
| Observability | Full | Good | Basic |
| Community | Large | Medium | Large |

## Observability

### Metrics (Prometheus)

```yaml
# Istio metrics automatically exported
# Example queries:

# Request rate by service
istio_requests_total{destination_service="reviews.default.svc.cluster.local"}

# Latency P99
histogram_quantile(0.99,
  sum(rate(istio_request_duration_milliseconds_bucket[5m]))
  by (le, destination_service)
)

# Error rate
sum(rate(istio_requests_total{response_code=~"5.*"}[5m]))
/ sum(rate(istio_requests_total[5m]))
```

### Distributed Tracing (Jaeger)

```yaml
# Istio tracing configuration
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    enableTracing: true
    defaultConfig:
      tracing:
        sampling: 100.0
        zipkin:
          address: jaeger-collector.istio-system:9411
```

### Kiali Dashboard

```bash
# Access Kiali
kubectl port-forward svc/kiali -n istio-system 20001:20001

# Kiali shows:
# - Service topology
# - Health status
# - Traffic flow
# - Istio configuration
```

## Gateway API

### Ingress Gateway

```yaml
# Istio Gateway for external traffic
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: my-tls-secret
      hosts:
        - "*.example.com"
```

### Kubernetes Gateway API

```yaml
# New Gateway API standard
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: istio
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: my-tls-secret
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
    - name: my-gateway
  hostnames:
    - "api.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api-service
          port: 80
```

## Best Practices

### 1. Gradual Adoption

```
Phase 1: Observability only
- Install mesh without strict mTLS
- Monitor metrics and traces
- Analyze service topology

Phase 2: Security
- Enable mTLS in permissive mode
- Gradual transition to strict mode
- Configure authorization policies

Phase 3: Traffic Management
- Canary deployments
- Circuit breakers
- Rate limiting
```

### 2. Resource Optimization

```yaml
# Optimize sidecar resources
apiVersion: v1
kind: Pod
metadata:
  annotations:
    sidecar.istio.io/proxyCPU: "100m"
    sidecar.istio.io/proxyMemory: "128Mi"
    sidecar.istio.io/proxyCPULimit: "500m"
    sidecar.istio.io/proxyMemoryLimit: "256Mi"
```

### 3. Excluding Services

```yaml
# Exclude namespace from mesh
apiVersion: v1
kind: Namespace
metadata:
  name: legacy-apps
  labels:
    istio-injection: disabled
```

```yaml
# Exclude specific pod
apiVersion: v1
kind: Pod
metadata:
  annotations:
    sidecar.istio.io/inject: "false"
```

### 4. Mesh Health Monitoring

```yaml
# Prometheus alerts for mesh
groups:
  - name: istio-alerts
    rules:
      - alert: IstioHighErrorRate
        expr: |
          sum(rate(istio_requests_total{response_code=~"5.*"}[5m]))
          / sum(rate(istio_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate in service mesh"

      - alert: IstioHighLatency
        expr: |
          histogram_quantile(0.99,
            sum(rate(istio_request_duration_milliseconds_bucket[5m])) by (le)
          ) > 1000
        for: 5m
        labels:
          severity: warning
```

## Anti-patterns

### 1. Big Bang Migration
```
❌ Enable mesh for all services at once
✅ Gradual adoption by namespace
```

### 2. Ignoring Overhead
```
❌ Not accounting for sidecar latency (1-5ms)
✅ Measure and monitor overhead
```

### 3. Overengineering
```
❌ Use all mesh features at once
✅ Enable features as needed
```

### 4. No Escape Hatch
```
❌ No plan to rollback mesh
✅ Document disabling procedure
```

## Troubleshooting

### Configuration Validation

```bash
# Istio configuration analysis
istioctl analyze

# Check sidecar injection
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].name}' | tr ' ' '\n' | grep istio

# Check mTLS status
istioctl x authz check <pod-name>
```

### Debug Envoy

```bash
# Envoy logs
kubectl logs <pod> -c istio-proxy

# Envoy admin interface
kubectl port-forward <pod> 15000:15000
# http://localhost:15000/config_dump

# Envoy statistics
istioctl proxy-status
```

## References

- [Istio Documentation](https://istio.io/latest/docs/)
- [Linkerd Documentation](https://linkerd.io/2.14/overview/)
- [Envoy Proxy](https://www.envoyproxy.io/docs/envoy/latest/)
- [SMI Specification](https://smi-spec.io/)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [CNCF Service Mesh Landscape](https://landscape.cncf.io/card-mode?category=service-mesh)
