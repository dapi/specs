# SRP-005 Load Balancing

## Определение

Load Balancing - распределение входящего сетевого трафика между несколькими экземплярами сервиса для оптимизации ресурсов, максимизации throughput, минимизации latency и обеспечения отказоустойчивости.

## Типы балансировки

### 1. DNS Round Robin
```
Простое переключение между IP-адресами
Уровень: DNS
Плюсы: Просто, не требует дополнительного ПО
Минусы: Нет health checks, кеширование DNS проблематично
```

### 2. Layer 4 (L4) - Transport Layer
```
Работает с TCP/UDP, по IP и порту
Примеры: HAProxy (TCP mode), NLB (AWS), Cloudflare Spectrum
Плюсы: Быстрый, эффективный
Минусы: Не видит содержимого запроса

Пример HAProxy:
listen app
    bind *:443
    mode tcp
    server app1 10.0.1.10:443 check
    server app2 10.0.1.11:443 check
```

### 3. Layer 7 (L7) - Application Layer
```
Работает с содержимым запроса (HTTP, HTTPS)
Примеры: ALB (AWS), Nginx, HAProxy (HTTP mode)
Плюсы: Может маршрутизировать по URL, headers, cookies
Минусы: Медленнее L4, больше ресурсов ЦП
```

## Алгоритмы балансировки

### 1. Round Robin
```
Запрос 1 → Server 1
Запрос 2 → Server 2
Запрос 3 → Server 3
Запрос 4 → Server 1

Работает когда: Все серверы равны по мощности
```

### 2. Least Connections
```
Новый запрос → Сервер с наименьшим количеством активных соединений
Работает когда: Запросы имеют разную продолжительность
```

### 3. Least Response Time
```
Новый запрос → Сервер с наименьшим средним временем ответа
Работает когда: Нужна адаптация под производительность
```

### 4. IP Hash
```
hash(client_ip) % number_of_servers = server_id
Работает когда: Нужна session affinity (sticky sessions)
```

### 5. Weighted (Взвешенный)
```
Server 1: weight 3
Server 2: weight 1
Server 3: weight 2

Распределение: 3:1:2

Работает когда: Серверы имеют разную производительность
```

### 6. Random with Two Choices
```
Выбираем случайно 2 сервера
Отправляем запрос на сервер с меньшей нагрузкой

Работает когда: Нужна простота и хорошее распределение
```

## Health Checks

### HTTP Health Check
```json
{
  "check": {
    "http": {
      "url": "http://server:8080/health",
      "timeout": "5s",
      "interval": "10s"
    }
  }
}
```

### TCP Health Check
```json
{
  "check": {
    "tcp": {
      "address": "server:8080",
      "timeout": "3s",
      "interval": "5s"
    }
  }
}
```

### Критерии unhealthy
* 3 consecutive failures
* Response time > 5 seconds
* Connection refused
* 5xx errors

## Сессионная привязность (Session Affinity)

Способность балансировщика направлять последующие запросы того же клиента на тот же сервер.

```yaml
# Nginx sticky sessions
upstream backend {
    ip_hash;  # Использовать IP адрес клиента

    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}

# Или с куками
upstream backend {
    sticky cookie srv_id expires=1h domain=.example.com path=/;

    server backend1.example.com;
    server backend2.example.com;
}
```

⚠️ **НЕ рекомендуется** - нарушает stateless архитектуру. Вместо этого использовать:
* Session storage (Redis)
* JWT токены
* Centralized sessions

## Конфигурация

### Nginx
```nginx
http {
    upstream app {
        least_conn;  # Или round_robin, ip_hash

        server app1:8080 max_fails=3 fail_timeout=30s weight=3;
        server app2:8080 max_fails=3 fail_timeout=30s weight=2;
        server app3:8080 max_fails=3 fail_timeout=30s backup;  # Только если все основные упали

        keepalive 32;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://app;
            proxy_connect_timeout 5s;
            proxy_send_timeout 10s;
            proxy_read_timeout 10s;
        }

        location /api {
            limit_req zone=api burst=20 nodelay;  # Rate limiting
            proxy_pass http://app;
        }
    }
}
```

### HAProxy
```haproxy
backend app
    balance leastconn  # Или roundrobin, source (IP hash)

    # Health checks
    option httpchk GET /health
    http-check expect status 200
    default-server inter 10s fall 3 rise 2

    server app1 10.0.1.10:8080 check weight 3
    server app2 10.0.1.11:8080 check weight 2
    server app3 10.0.1.12:8080 check weight 2

    # Параметры
    timeout connect 5s
    timeout server 60s
    timeout check 5s
```

### Kubernetes Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer

  # Session affinity (НЕ рекомендуется)
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

## Проблемы и решения

### Thundering Herd

Проблема: Все серверы стартуют одновременно, балансировщик перегружен

Решение: Gradual rollout, connection rate limiting

### Uneven Load

Проблема: Неравномерная нагрузка из-за long-lived connections

Решение: Least connections algorithm, keep alive timeout

### Health Check Flooding

Проблема: Балансировщик слишком часто проверяет health

Решение: Настроить интервал 10-30s, использовать shared health state

### Connection Draining

При остановке сервера нужно время, чтобы завершить активные запросы:

```haproxy
server app1 10.0.1.10:8080 check weight 3
    # Дождаться завершения в течение 30s
    # Перестать отправлять НОВЫЕ запросы
    # После 30s закрыть соединения
```

## Multizone / Geographic Balancing

### Active-Active
```
┌─────────────┐
│ DNS / GSLB  │
└──────┬──────┘
       │
┌──────▼────────────┬─────────────┐
│ Region 1          │ Region 2    │
│ ┌────────────┐    │ ┌──────────┐│
│ │ Load       │    │ │ Load     ││
│ │ Balancer   │    │ │Balancer  ││
│ └──────┬─────┘    │ └─────┬────┘│
│        │          │       │     │
│   ┌────▼─────┐    │  ┌────▼───┐│
│   │ Server   │    │  │ Server ││
│   │          │    │  │        ││
│   └──────────┘    │  └────────┘│
└───────────────────┴────────────┘
```

### Active-Passive
```
Primary Region (100% traffic)
        │
      Failover
        │
        ▼
Backup Region (0% traffic, hot standby)
```

## Метрики

```
load_balancer_requests_total (counter)
load_balancer_active_connections (gauge)
load_balancer_backend_up (gauge)
load_balancer_request_duration_seconds (histogram)
load_balancer_backend_errors_total (counter)
```

## Конфигурация через переменные окружения

```
# Алгоритм
LB_ALGORITHM=least_conn  # round_robin, least_conn, ip_hash

# Параметры
LB_HEALTH_CHECK_INTERVAL=10s
LB_HEALTH_CHECK_TIMEOUT=5s
LB_MAX_FAILS=3
LB_FAIL_TIMEOUT=30s

# Timeouts
LB_CONNECT_TIMEOUT=5s
LB_SEND_TIMEOUT=10s
LB_READ_TIMEOUT=10s

# Rate limiting
LB_RATE_LIMIT_ENABLED=true
LB_RATE_LIMIT_RPS=1000
LB_RATE_LIMIT_BURST=200
```

## Best practices

✅ **Делать**
* Health checks на всех backend-серверах
* Использовать least connections для неравных запросов
* Настраивать timeouts
* Log failed requests
* Мониторить backend health
* Иметь backup-серверы

❌ **Не делать**
* IP Hash без необходимости
* Sticky sessions (при наличии альтернатив)
* Отправлять трафик на unhealthy серверы
* Игнорировать настройку timeouts
* Health check слишком часто или редко

## Дополнительные ресурсы

* [HAProxy Documentation](https://www.haproxy.org/documentation/)
* [NGINX Load Balancing](https://nginx.org/en/docs/http/load_balancing.html)
* [AWS Load Balancer Types](https://aws.amazon.com/elasticloadbalancing/)
* [Kubernetes Service Types](https://kubernetes.io/docs/concepts/services-networking/service/)
