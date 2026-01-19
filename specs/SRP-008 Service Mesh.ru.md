# SRP-008 Service Mesh

Service Mesh — инфраструктурный слой для управления коммуникацией между микросервисами, обеспечивающий наблюдаемость, безопасность и контроль трафика.

## Зачем нужен Service Mesh?

### Проблемы микросервисной архитектуры
- Сложность управления межсервисной коммуникацией
- Отсутствие единых политик безопасности
- Трудности с трассировкой и дебагом
- Разнородная реализация retry, timeout, circuit breaker

### Преимущества Service Mesh
- **Наблюдаемость**: автоматический сбор метрик, трейсов, логов
- **Безопасность**: mTLS между сервисами из коробки
- **Контроль трафика**: canary releases, A/B тестирование, fault injection
- **Политики**: rate limiting, access control на уровне инфраструктуры

## Архитектура Service Mesh

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
# Pod с sidecar proxy
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
    # Sidecar injected автоматически
    # - name: istio-proxy
    #   image: envoyproxy/envoy
```

## Istio

### Установка

```bash
# Установка istioctl
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Установка Istio с demo профилем
istioctl install --set profile=demo -y

# Включение автоматической инъекции sidecar
kubectl label namespace default istio-injection=enabled
```

### Профили установки

| Профиль | Описание | Компоненты |
|---------|----------|------------|
| minimal | Минимальная установка | istiod |
| default | Production рекомендуемый | istiod, ingress gateway |
| demo | Демонстрация всех функций | Все компоненты |
| empty | Ничего не устанавливается | Пустой |

### Traffic Management

#### VirtualService

```yaml
# Маршрутизация трафика
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
# Определение версий сервиса
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
# 90% на v1, 10% на v2
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

#### Timeout и Retry

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
# Strict mTLS для namespace
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
# Разрешить доступ только от фронтенда
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

### Установка

```bash
# Установка CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

# Проверка требований
linkerd check --pre

# Установка control plane
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# Проверка установки
linkerd check
```

### Аннотации для инъекции

```yaml
# Включение Linkerd для deployment
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
# Определение маршрутов для метрик
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
# Canary deployment с Linkerd
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

## Сравнение Service Mesh

| Характеристика | Istio | Linkerd | Consul Connect |
|----------------|-------|---------|----------------|
| Proxy | Envoy | linkerd2-proxy | Envoy |
| Сложность | Высокая | Низкая | Средняя |
| Ресурсы | Много | Мало | Средне |
| mTLS | Да | Да | Да |
| Traffic Management | Полный | Базовый | Средний |
| Multi-cluster | Да | Да | Да |
| Observability | Полный | Хороший | Базовый |
| Сообщество | Большое | Среднее | Большое |

## Observability

### Метрики (Prometheus)

```yaml
# Istio метрики автоматически экспортируются
# Пример запросов:

# Request rate по сервису
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
# Istio трейсинг конфигурация
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
# Доступ к Kiali
kubectl port-forward svc/kiali -n istio-system 20001:20001

# Kiali показывает:
# - Топологию сервисов
# - Health status
# - Traffic flow
# - Конфигурацию Istio
```

## Gateway API

### Ingress Gateway

```yaml
# Istio Gateway для внешнего трафика
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
# Новый стандарт Gateway API
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

### 1. Постепенное внедрение

```
Этап 1: Observability only
- Установка mesh без strict mTLS
- Мониторинг метрик и трейсов
- Анализ топологии сервисов

Этап 2: Security
- Включение mTLS в permissive mode
- Постепенный переход на strict mode
- Настройка authorization policies

Этап 3: Traffic Management
- Canary deployments
- Circuit breakers
- Rate limiting
```

### 2. Оптимизация ресурсов

```yaml
# Оптимизация sidecar ресурсов
apiVersion: v1
kind: Pod
metadata:
  annotations:
    sidecar.istio.io/proxyCPU: "100m"
    sidecar.istio.io/proxyMemory: "128Mi"
    sidecar.istio.io/proxyCPULimit: "500m"
    sidecar.istio.io/proxyMemoryLimit: "256Mi"
```

### 3. Исключение сервисов

```yaml
# Исключение namespace из mesh
apiVersion: v1
kind: Namespace
metadata:
  name: legacy-apps
  labels:
    istio-injection: disabled
```

```yaml
# Исключение конкретного pod
apiVersion: v1
kind: Pod
metadata:
  annotations:
    sidecar.istio.io/inject: "false"
```

### 4. Мониторинг здоровья mesh

```yaml
# Prometheus алерты для mesh
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

## Антипаттерны

### 1. Big Bang Migration
```
❌ Включить mesh для всех сервисов сразу
✅ Постепенное внедрение по namespace
```

### 2. Игнорирование overhead
```
❌ Не учитывать latency от sidecar (1-5ms)
✅ Измерять и мониторить overhead
```

### 3. Overengineering
```
❌ Использовать все функции mesh сразу
✅ Включать функции по мере необходимости
```

### 4. Отсутствие escape hatch
```
❌ Нет плана отката mesh
✅ Документировать процедуру отключения
```

## Troubleshooting

### Проверка конфигурации

```bash
# Istio configuration analysis
istioctl analyze

# Проверка sidecar injection
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].name}' | tr ' ' '\n' | grep istio

# Проверка mTLS статуса
istioctl x authz check <pod-name>
```

### Debug Envoy

```bash
# Логи Envoy
kubectl logs <pod> -c istio-proxy

# Envoy admin interface
kubectl port-forward <pod> 15000:15000
# http://localhost:15000/config_dump

# Статистика Envoy
istioctl proxy-status
```

## Ссылки

- [Istio Documentation](https://istio.io/latest/docs/)
- [Linkerd Documentation](https://linkerd.io/2.14/overview/)
- [Envoy Proxy](https://www.envoyproxy.io/docs/envoy/latest/)
- [SMI Specification](https://smi-spec.io/)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [CNCF Service Mesh Landscape](https://landscape.cncf.io/card-mode?category=service-mesh)
