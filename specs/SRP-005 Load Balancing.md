# SRP-005 Load Balancing

## Definition

Load Balancing is the distribution of incoming network traffic between multiple service instances to optimize resources, maximize throughput, minimize latency, and provide fault tolerance.

## Load balancing types

### 1. DNS Round Robin
```
Simple rotation between IP addresses
Level: DNS
Pros: Simple, doesn't require additional software
Cons: No health checks, DNS caching problematic
```

### 2. Layer 4 (L4) - Transport Layer
```
Works with TCP/UDP, by IP and port
Examples: HAProxy (TCP mode), NLB (AWS), Cloudflare Spectrum
Pros: Fast, efficient
Cons: Doesn't see request content

HAProxy example:
listen app
    bind *:443
    mode tcp
    server app1 10.0.1.10:443 check
    server app2 10.0.1.11:443 check
```

### 3. Layer 7 (L7) - Application Layer
```
Works with request content (HTTP, HTTPS)
Examples: ALB (AWS), Nginx, HAProxy (HTTP mode)
Pros: Can route by URL, headers, cookies
Cons: Slower than L4, more CPU resources
```

## Balancing algorithms

### 1. Round Robin
```
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1

Works when: All servers have equal capacity
```

### 2. Least Connections
```
New request → Server with fewest active connections
Works when: Requests have different duration
```

### 3. Least Response Time
```
New request → Server with lowest average response time
Works when: Need to adapt to performance
```

### 4. IP Hash
```
hash(client_ip) % number_of_servers = server_id
Works when: Need session affinity (sticky sessions)
```

### 5. Weighted
```
Server 1: weight 3
Server 2: weight 1
Server 3: weight 2

Distribution: 3:1:2

Works when: Servers have different performance
```

### 6. Random with Two Choices
```
Randomly select 2 servers
Send request to server with less load

Works when: Need simplicity and good distribution
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

### Unhealthy criteria
* 3 consecutive failures
* Response time > 5 seconds
* Connection refused
* 5xx errors

## Session Affinity

Ability of load balancer to direct subsequent requests from the same client to the same server.

```yaml
# Nginx sticky sessions
upstream backend {
    ip_hash;  # Use client IP address

    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}

# Or with cookies
upstream backend {
    sticky cookie srv_id expires=1h domain=.example.com path=/;

    server backend1.example.com;
    server backend2.example.com;
}
```

⚠️ **NOT recommended** - violates stateless architecture. Instead use:
* Session storage (Redis)
* JWT tokens
* Centralized sessions

## Configuration

### Nginx
```nginx
http {
    upstream app {
        least_conn;  # Or round_robin, ip_hash

        server app1:8080 max_fails=3 fail_timeout=30s weight=3;
        server app2:8080 max_fails=3 fail_timeout=30s weight=2;
        server app3:8080 max_fails=3 fail_timeout=30s backup;  # Only if all main servers are down

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
    balance leastconn  # Or roundrobin, source (IP hash)

    # Health checks
    option httpchk GET /health
    http-check expect status 200
    default-server inter 10s fall 3 rise 2

    server app1 10.0.1.10:8080 check weight 3
    server app2 10.0.1.11:8080 check weight 2
    server app3 10.0.1.12:8080 check weight 2

    # Parameters
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

  # Session affinity (NOT recommended)
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

## Problems and solutions

### Thundering Herd

Problem: All servers start simultaneously, load balancer is overloaded

Solution: Gradual rollout, connection rate limiting

### Uneven Load

Problem: Uneven load due to long-lived connections

Solution: Least connections algorithm, keep alive timeout

### Health Check Flooding

Problem: Load balancer checks health too frequently

Solution: Configure interval 10-30s, use shared health state

### Connection Draining

When stopping a server, need time to complete active requests:

```haproxy
server app1 10.0.1.10:8080 check weight 3
    # Wait for completion for 30s
    # Stop sending NEW requests
    # After 30s close connections
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

## Metrics

```
load_balancer_requests_total (counter)
load_balancer_active_connections (gauge)
load_balancer_backend_up (gauge)
load_balancer_request_duration_seconds (histogram)
load_balancer_backend_errors_total (counter)
```

## Configuration via environment variables

```
# Algorithm
LB_ALGORITHM=least_conn  # round_robin, least_conn, ip_hash

# Parameters
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

✅ **Do**
* Health checks on all backend servers
* Use least connections for requests of unequal duration
* Configure timeouts
* Log failed requests
* Monitor backend health
* Have backup servers

❌ **Don't**
* IP Hash without necessity
* Sticky sessions (if alternatives exist)
* Send traffic to unhealthy servers
* Ignore timeout configuration
* Health check too frequently or infrequently

## Additional resources

* [HAProxy Documentation](https://www.haproxy.org/documentation/)
* [NGINX Load Balancing](https://nginx.org/en/docs/http/load_balancing.html)
* [AWS Load Balancer Types](https://aws.amazon.com/elasticloadbalancing/)
* [Kubernetes Service Types](https://kubernetes.io/docs/concepts/services-networking/service/)
