# SRP-007 API Gateway (API шлюз)

API Gateway - это паттерн проектирования, который служит единой точкой входа для клиентских запросов, предоставляя маршрутизацию, аутентификацию, ограничение скорости, кеширование и другие сквозные возможности для микросервисной архитектуры.

---

## Что такое API Gateway?

```
Без API Gateway:
┌─────────────┐
│  Frontend   │
└──────┬──────┘
       │
       ├──→ [Service 1] → Database 1
       ├──→ [Service 2] → Database 2
       ├──→ [Service 3] → Database 3
       └──→ [Service 4] → Database 4

С API Gateway:
┌─────────────┐
│  Frontend   │
└──────┬──────┘
       │
       ↓
┌──────────────┐
│ API Gateway  │ ← Auth, Rate Limiting, Cache
└──────┬───────┘
       │
       ├──→ [Service 1]
       ├──→ [Service 2]
       ├──→ [Service 3]
       └──→ [Service 4]
```

**Цели API Gateway:**
- Единая точка входа для клиентов
- Centralized security and auth
- Rate limiting and throttling
- Request/response transformation
- Service discovery and routing
- Monitoring and logging
- API versioning and composition

---

## Ошибки проектирования ❌

### ❌ API Gateway как монолит
```
Вся логика в одном месте:
- Auth
- Business logic
- Data aggregation
- Form validation

Приводит к: God Object, сложность, единая точка отказа
```

### ❌ Сетевая связность (Chatty API Gateway)
```
Клиент: GET /dashboard
↓
API Gateway:
  ├→ GET /users/123
  ├→ GET /orders?user=123
  ├→ GET /payments?user=123
  └→ GET /recommendations?user=123
↓
Ответ (составлен из 4 ответов): 800ms latency

Проблема: Синхронные вызовы, медленный response
```

### ❌ Сохранение состояния
```python
# В gateway храним сессии и данные
# Нарушает stateless принцип
# Усложняет масштабирование
session_store = Redis()
session_store.set(f"user:{user_id}", session_data)
```

### ❌ Жесткая связанность
```yaml
# Сервисы напрямую зависят от Gateway
# Невозможно обратиться напрямую в тестах
services:
  - name: order-service
    only_accessible_through: api-gateway
```

---

## Архитектура API Gateway

### Layer 1: Edge Layer (входящий трафик)

```yaml
edge_layer:
  components:
    load_balancer:
      type: AWS ALB / NGINX / HAProxy
      purpose: Distribute traffic
      ssl_termination: true

    cdn:
      provider: Cloudflare / CloudFront / Fastly
      caching: static_assets
      ddos_protection: true

    waf:
      provider: AWS WAF / Cloudflare WAF
      rules:
        - sql_injection_block
        - xss_block
        - rate_limit_by_ip
        - geo_block
```

### Layer 2: API Gateway (core)

```yaml
core_gateway:
  responsibilities:
    authentication:
      - jwt_validation
      - api_key_validation
      - oauth2_introspection

    authorization:
      - rbac_check
      - rate_limit_check
      - quota_check

    routing:
      - path_based: /api/v1/users → user-service
      - header_based: X-API-Version → different versions
      - method_based: GET / POST / PUT / DELETE

    transformation:
      - request_enrichment: Add correlation_id
      - response_format: JSON / XML transformation
      - header_manipulation: Add security headers

    observability:
      - logging: All requests/responses
      - metrics: Latency, error rate, throughput
      - tracing: Forward trace_id to services
```

### Layer 3: Service Mesh (optional)

```yaml
service_mesh:
  istio:
    traffic_management:
      - load_balancing
      - circuit_breaking
      - retry_policies
      - timeouts

    security:
      - mTLS
      - authorization_policies
      - encryption

    observability:
      - distributed_tracing
      - metrics_collection
      - service_discovery
```

---

## Типовые функции API Gateway

### 1. Аутентификация и авторизация

```python
class APIGateway:
    def authenticate_request(self, request):
        """Аутентификация запроса"""
        auth_header = request.headers.get('Authorization')

        if not auth_header:
            return {'error': 'Missing Authorization header'}, 401

        # JWT validation
        if auth_header.startswith('Bearer '):
            token = auth_header.replace('Bearer ', '')
            payload = self.validate_jwt(token)

            if not payload:
                return {'error': 'Invalid token'}, 401

            request.user_id = payload['sub']
            request.scopes = payload['scopes']

        # API Key validation
        elif auth_header.startswith('Basic '):
            api_key = self.extract_api_key(auth_header)
            if not self.validate_api_key(api_key):
                return {'error': 'Invalid API key'}, 401

            request.client_id = self.get_client_id(api_key)

        return request

    def authorize_request(self, request, route):
        """Авторизация на основе RBAC"""
        required_scopes = route['required_scopes']

        if not any(scope in request.scopes for scope in required_scopes):
            return {'error': 'Insufficient permissions'}, 403

        return request

# Использование
@app.route('/api/v1/users')
def users(request):
    # Auth middleware
    auth_result, status = gateway.authenticate_request(request)
    if status == 401:
        return auth_result, status

    # Authorization
    route = {'required_scopes': ['users.read']}
    auth_result, status = gateway.authorize_request(request, route)
    if status == 403:
        return auth_result, status

    # Forward to service
    return forward_to_user_service(request)
```

### 2. Rate Limiting

```python
class RateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client

    def is_allowed(self, request, route):
        """Проверяем лимит запросов"""
        # Key: user + route
        key = f"rate_limit:{request.user_id}:{route['path']}"

        # Limits
        limit = route.get('rate_limit', 100)
        window = route.get('window_seconds', 3600)

        current = self.redis.incr(key)
        if current == 1:
            self.redis.expire(key, window)

        if current > limit:
            return False, {
                'error': 'Rate limit exceeded',
                'limit': limit,
                'window_seconds': window,
                'retry_after': self.redis.ttl(key)
            }

        return True, {'remaining': limit - current}

# Использование
@app.route('/api/v1/search')
def search(request):
    allowed, result = gateway.rate_limiter.is_allowed(
        request,
        route={'path': '/api/v1/search', 'rate_limit': 1000}
    )

    if not allowed:
        request.headers['Retry-After'] = str(result['retry_after'])
        return result['error'], 429

    return forward_to_search_service(request)
```

### 3. Request Routing

```python
class Router:
    def __init__(self):
        self.routes = {
            '/api/v1/users': {'service': 'user-service', 'version': 'v1'},
            '/api/v1/orders': {'service': 'order-service', 'version': 'v1'},
            '/api/v2/users': {'service': 'user-service-v2', 'version': 'v2'},
            '/api/v1/payments': {'service': 'payment-service', 'version': 'v1'},
        }
    def route_request(self, request):
        """Маршрутизируем запрос к нужному сервису"""
        path = request.path

        # Exact match
        if path in self.routes:
            route = self.routes[path]
            return self.forward_to_service(request, route)

        # Prefix match
        for route_path, route_config in self.routes.items():
            if path.startswith(route_path):
                return self.forward_to_service(request, route_config)

        # Not found
        return {'error': 'Not found'}, 404

    def forward_to_service(self, request, route):
        """Передаем запрос во внутренний сервис"""
        service = route['service']

        # Service discovery
        service_url = self.discovery.get_service_url(service)

        # Copy headers
        headers = {k: v for k, v in request.headers.items()}
        headers['X-Correlation-ID'] = request.correlation_id
        headers['X-Forwarded-For'] = request.client_ip

        # Forward
        response = requests.request(
            method=request.method,
            url=f"{service_url}{request.path}",
            headers=headers,
            json=request.body,
            timeout=30
        )

        return response.json(), response.status_code
```

### 4. Request/Response Transformation

```python
class Transformer:
    def transform_request(self, request, route):
        """Enrich request before forwarding"""
        transformations = route.get('request_transformations', [])

        for transform in transformations:
            if transform == 'add_correlation_id':
                request.headers['X-Correlation-ID'] = generate_uuid()

            elif transform == 'add_user_context':
                request.headers['X-User-ID'] = request.user_id
                request.headers['X-User-Role'] = request.user_role

            elif transform == 'add_timestamp':
                request.headers['X-Request-Timestamp'] = str(time.time())

        return request

    def transform_response(self, response, route):
        """Transform response before returning to client"""
        # Remove sensitive headers
        response.headers.pop('X-Service-Version', None)
        response.headers.pop('Server', None)

        # Format response
        if route.get('format_response'):
            response.body = {
                'data': response.body,
                'metadata': {
                    'version': 'v1',
                    'timestamp': time.time(),
                    'request_id': response.request_id
                }
            }

        return response
```

### 5. API Composition (Backend for Frontend - BFF)

```python
class APIComposer:
    def compose_dashboard(self, request, user_id):
        """Компоновка данных из нескольких сервисов"""

        # Параллельные вызовы
        user_future = self.async_get(f"/users/{user_id}")
        orders_future = self.async_get(f"/orders?user={user_id}")
        payments_future = self.async_get(f"/payments?user={user_id}")

        # Ожидание всех ответов
        user = user_future.result()
        orders = orders_future.result()
        payments = payments_future.result()

        # Compose response
        return {
            'user': user,
            'stats': {
                'total_orders': len(orders),
                'total_payments': sum(p['amount'] for p in payments)
            },
            'recent_orders': orders[:5],
            'insights': self.generate_insights(orders, payments)
        }

    def generate_insights(self, orders, payments):
        """Генерация insights на основе данных"""
        return {
            'avg_order_value': self.calculate_avg(orders),
            'preferred_payment_method': self.most_common(payments, 'method')
        }
```

---

## Платформы API Gateway

### AWS API Gateway

```yaml
# serverless.yml
service: my-api

provider:
  name: aws
  runtime: python3.9

functions:
  user-api:
    handler: handler.users
    events:
      - http:
          path: /users/{id}
          method: get
          authorizer: ${self:custom.authorizer}
          cors: true

  order-api:
    handler: handler.orders
    events:
      - http:
          path: /orders
          method: post
          authorizer: ${self:custom.authorizer}
          cors: true

custom:
  authorizer:
    name: custom-authorizer
    type: request
    identitySource: method.request.header.Authorization
```

### Kong

```yaml
# kong.yml
services:
  - name: user-service
    url: http://user-service:8080
    routes:
      - name: users
        paths: [/api/v1/users]
        methods: [GET, POST, PUT, DELETE]
      - name: users-by-id
        paths: [/api/v1/users/{id}]
        methods: [GET, PUT, DELETE]
    plugins:
      - name: jwt
        config:
          secret_is_base64: false
          run_on_preflight: false
      - name: rate-limiting
        config:
          minute: 1000
          policy: redis
          redis_host: redis

  - name: order-service
    url: http://order-service:8080
    routes:
      - name: orders
        paths: [/api/v1/orders]
        methods: [GET, POST]
        plugins:
          - name: key-auth
          - name: acl
            config:
              allow: [admin, customer]
```

### NGINX (дополнительно Lua)

```nginx
# nginx.conf
http {
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=payment:10m rate=1r/s;

    upstream user_service {
        server user-service-1:8080;
        server user-service-2:8080;
    }

    upstream order_service {
        server order-service-1:8080;
        server order-service-2:8080;
    }

    server {
        listen 80;
        server_name api.example.com;

        location /api/v1/users {
            limit_req zone=api burst=20 nodelay;
            auth_request /auth;

            proxy_pass http://user_service;
            proxy_set_header X-User-ID $user_id;
            proxy_set_header X-Correlation-ID $request_id;
        }

        location /api/v1/payments {
            limit_req zone=payment burst=5 nodelay;
            auth_request /auth;

            proxy_pass http://payment_service;
            proxy_set_header X-User-ID $user_id;
        }

        location = /auth {
            internal;
            proxy_pass http://auth-service/auth/verify;
            proxy_pass_request_body off;
            proxy_set_header Content-Length "";
            proxy_set_header X-Original-URI $request_uri;
        }
    }
}
```

### Express.js API Gateway

```javascript
const express = require('express');
const httpProxy = require('http-proxy-middleware');
const jwt = require('jsonwebtoken');

const gateway = express();

// Authentication middleware
const authenticate = (req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};

// Rate limiting
const rateLimit = require('express-rate-limit');
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: 'Too many requests from this IP'
});

// Routes
const userProxy = httpProxy({
  target: process.env.USER_SERVICE_URL,
  changeOrigin: true,
  pathRewrite: {
    '^/api/v1/users': '/'
  },
  onProxyReq: (proxyReq, req) => {
    proxyReq.setHeader('X-User-ID', req.user.id);
    proxyReq.setHeader('X-Correlation-ID', req.correlationId);
  }
});

const orderProxy = httpProxy({
  target: process.env.ORDER_SERVICE_URL,
  changeOrigin: true,
  pathRewrite: {
    '^/api/v1/orders': '/'
  }
});

// Apply middlewares
gateway.use(limiter);
gateway.use(authenticate);

// Routes
gateway.use('/api/v1/users', userProxy);
gateway.use('/api/v1/orders', orderProxy);

// BFF (Backend for Frontend) - composition
gateway.get('/api/v1/dashboard', async (req, res) => {
  try {
    const [user, orders] = await Promise.all([
      axios.get(`${process.env.USER_SERVICE_URL}/users/${req.user.id}`),
      axios.get(`${process.env.ORDER_SERVICE_URL}/orders?user=${req.user.id}`)
    ]);

    res.json({
      user: user.data,
      stats: {
        totalOrders: orders.data.length
      },
      orders: orders.data
    });
  } catch (error) {
    res.status(500).json({ error: 'Failed to fetch dashboard data' });
  }
});

gateway.listen(3000);
```

---

## Метрики API Gateway

```python
GATEWAY_METRICS = {
    # Трафик
    'gateway_requests_total': 'Общее количество запросов',
    'gateway_requests_by_route': 'Запросы по маршруту',
    'gateway_requests_by_method': 'Запросы по HTTP методу',

    # Производительность
    'gateway_request_duration': 'Время обработки запроса',
    'gateway_upstream_latency': 'Время ответа backend сервиса',
    'gateway_proxy_latency': 'Время работы прокси',

    # Ошибки
    'gateway_errors_total': 'Общее количество ошибок',
    'gateway_errors_by_type': 'Ошибки по типу (4xx, 5xx)',
    'gateway_timeout_errors': 'Таймауты',

    # Rate limiting
    'gateway_rate_limit_hits_total': 'Срабатываний rate limit',
    'gateway_rate_limit_remaining': 'Оставшихся запросов',

    # Состояние
    'gateway_active_connections': 'Активных соединений',
    'gateway_waiting_requests': 'Запросов в очереди',
    'gateway_services_healthy': 'Здоровых backend сервисов',

    # Кэширование
    'gateway_cache_hits_total': 'Попаданий в кэш',
    'gateway_cache_misses_total': 'Промахов кэша',
    'gateway_cache_size_bytes': 'Размер кэша'
}
```

---

## Сравнение решений

| Решение | Управление | Features | Cost | Self-Hosted | Best For |
|---------|-----------|----------|------|-------------|----------|
| AWS API Gateway | Провайдер | Full | $$ | No | AWS workloads |
| Kong | OSS / Enterprise | Extensible | $ / $$$ | Yes | Hybrid clouds |
| NGINX | OSS / Plus | Flexible | $ / $$ | Yes | Simple routing |
| HAProxy | OSS / Enterprise | High performance | $ / $$$ | Yes | Load balancing |
| Express Gateway | OSS | Node.js | Free | Yes | Node.js teams |
| Netflix Zuul | OSS | Java | Free | Yes | Java/Spring |
| Ambassador | OSS / Enterprise | Kubernetes | $ / $$$ | Yes | Kubernetes |
| Tyk | OSS / Enterprise | API Management | $ / $$$ | Yes | Full lifecycle |

---

*API Gateway - паттерн единой точки входа для микросервисной архитектуры*
