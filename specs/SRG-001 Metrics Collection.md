# SRG-001 Metrics Collection

## Definition

Metrics are quantitative data about system behavior in production: number of requests, response time, resource usage, etc.

## Metric formats

### Counter

```
Always increases, never decreases
Used for: Number of requests, errors, completed tasks

Examples:
  http_requests_total{method="GET",status="200"} 12500
  http_requests_total{method="POST",status="500"} 5
```

### Gauge

```
Can increase and decrease
Used for: Current values (memory, connections, temperatures)

Examples:
  memory_usage_bytes 1073741824
  active_connections 23
  queue_length 5
```

### Histogram

```
Distribution of values (latency, sizes)
Used for: Response time, request sizes, processing time

Examples:
  http_request_duration_seconds_bucket{le="0.1"} 100
  http_request_duration_seconds_bucket{le="0.5"} 150
  http_request_duration_seconds_bucket{le="+Inf"} 200
  http_request_duration_seconds_sum 75.5
  http_request_duration_seconds_count 200
```

### Summary

```
Similar to histogram, but calculates quantiles
Used for: p50, p95, p99 latency

Examples:
  http_request_duration_seconds{quantile="0.5"} 0.05
  http_request_duration_seconds{quantile="0.95"} 0.15
  http_request_duration_seconds{quantile="0.99"} 0.25
  http_request_duration_seconds_sum 75.5
  http_request_duration_seconds_count 200
```

## What to collect

### Mandatory metrics

#### HTTP API

```
http_requests_total (counter)
  - method
  - path
  - status_code
  - service

http_request_duration_seconds (histogram)
  - method
  - path

http_request_size_bytes (histogram)
http_response_size_bytes (histogram)
```

#### Application

```
process_cpu_seconds_total (counter)
process_resident_memory_bytes (gauge)
process_virtual_memory_bytes (gauge)

threads_active (gauge)
goroutines_active (gauge)  # for Go

app_info (gauge)
  - version
  - build
  - environment
```

#### Resources

```
system_cpu_usage (gauge)
system_memory_usage_bytes (gauge)
system_memory_total_bytes (gauge)
system_disk_usage_bytes (gauge)
system_disk_total_bytes (gauge)
system_network_receive_bytes_total (counter)
system_network_transmit_bytes_total (counter)
```

#### Dependencies

```
database_connections_active (gauge)
database_connections_idle (gauge)
database_query_duration_seconds (histogram)
database_queries_total (counter)
  - operation
  - table

cache_hits_total (counter)
cache_misses_total (counter)
cache_duration_seconds (histogram)

http_client_requests_total (counter)
http_client_duration_seconds (histogram)
  - target_service
  - method
  - status_code
```

#### Business Metrics

```
orders_created_total (counter)
orders_completed_total (counter)
orders_failed_total (counter)
order_processing_duration_seconds (histogram)

payment_processed_total (counter)
  - currency
  - payment_method
payment_amount_usd_total (counter)
```

## Code instrumentation

### Python (FastAPI)

```python
from prometheus_client import Counter, Histogram, Gauge
import time

# Define metrics
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status_code']
)

REQUEST_DURATION = Histogram(
    'http_request_duration_seconds',
    'Request duration',
    ['method', 'endpoint']
)

ACTIVE_REQUESTS = Gauge(
    'http_requests_active',
    'Number of active requests'
)

@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    # Track active requests
    ACTIVE_REQUESTS.inc()

    start_time = time.time()

    try:
        response = await call_next(request)
        status_code = response.status_code
    except Exception as e:
        status_code = 500
        raise
    finally:
        # Track metrics
        REQUEST_COUNT.labels(
            method=request.method,
            endpoint=request.url.path,
            status_code=status_code
        ).inc()

        REQUEST_DURATION.labels(
            method=request.method,
            endpoint=request.url.path
        ).observe(time.time() - start_time)

        ACTIVE_REQUESTS.dec()

    return response
```

### Java (Spring Boot)

```java
@RestController
public class OrderController {

    private final Counter ordersCreated = Counter.builder("orders_created_total")
        .description("Total orders created")
        .register(Metrics.globalRegistry);

    private final Timer orderProcessingTime = Timer.builder("order_processing_duration")
        .description("Order processing time")
        .register(Metrics.globalRegistry);

    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
        // Track counter
        ordersCreated.increment();

        // Track timer
        return orderProcessingTime.record(() -> {
            // Business logic
            Order order = processOrder(request);
            return ResponseEntity.ok(order);
        });
    }
}

// Configuration
@Configuration
public class MetricsConfig {

    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config()
            .commonTags("application", "order-service")
            .commonTags("environment", System.getenv("ENVIRONMENT"));
    }
}
```

### Go

```go
package main

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "net/http"
    "time"
)

var (
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "path", "status"},
    )

    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "Duration of HTTP requests",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "path"},
    )

    activeRequests = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "http_requests_active",
            Help: "Number of active requests",
        },
    )
)

func metricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        activeRequests.Inc()
        defer activeRequests.Dec()

        // Wrap response writer to capture status code
        wrapped := &responseWriter{
            ResponseWriter: w,
            statusCode:     http.StatusOK,
        }

        next.ServeHTTP(wrapped, r)

        duration := time.Since(start).Seconds()
        httpRequestsTotal.WithLabelValues(
            r.Method,
            r.URL.Path,
            string(wrapped.statusCode),
        ).Inc()

        httpRequestDuration.WithLabelValues(
            r.Method,
            r.URL.Path,
        ).Observe(duration)
    })
}
```

## Labels (tags)

### Required labels

```
environment: production|staging|development
service: order-service|payment-service|user-service
team: platform|payments|growth
version: 1.2.3|2.0.0
region: us-east-1|eu-west-1
```

### Optional labels

```
customer_tier: free|basic|pro|enterprise
feature: checkout|reports|admin
deployment: blue|green|canary
```

⚠️ **Important**: Do not use high-cardinality labels:
❌ user_id, request_id, timestamp, ip_address

### Debugging

For temporary debugging, you can use:
```python
# Only in development
if os.getenv('ENVIRONMENT') == 'development':
    REQUEST_DURATION = Histogram(
        'http_request_duration_seconds',
        'Request duration',
        ['method', 'endpoint', 'user_id']  # Don't do this in production!
    )
```

## Metrics export

### Prometheus

```python
from prometheus_client import start_http_server

# Start metrics server
start_http_server(9090)

# Metrics available at http://localhost:9090/metrics
```

### StatsD

```python
from datadog import DogStatsd

statsd = DogStatsd(
    host='statsd.example.com',
    port=8125,
    namespace='app',
    constant_tags=['env:production', 'service:orders']
)

# Counter
statsd.increment('orders.created', tags=['currency:usd'])

# Gauge
statsd.gauge('queue.length', 23, tags=['queue:orders'])

# Histogram
statsd.histogram('order.processing_time', 0.5, tags=['priority:high'])
```

### OpenTelemetry

```python
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import (
    ConsoleMetricsExporter,
    PeriodicExportingMetricReader,
)

# Configure meter
provider = MeterProvider(
    metric_readers=[
        PeriodicExportingMetricReader(
            ConsoleMetricsExporter(),
            export_interval_millis=60000,
        )
    ],
)
metrics.set_meter_provider(provider)

meter = metrics.get_meter("order-service")

# Create counter
counter = meter.create_counter(
    "orders.created",
    description="Number of orders created",
)

counter.add(1, {"currency": "usd", "payment_method": "card"})
```

## Push vs Pull

### Pull (Prometheus)

```yaml
# Prometheus config
scrape_configs:
  - job_name: 'order-service'
    static_configs:
      - targets: ['order-service:8080']
    scrape_interval: 15s
    metrics_path: '/metrics'
```

Pros:
* Simple for Prometheus
* Service doesn't know where to send
* Centralized management

Cons:
* Not suitable for all metrics
* Requires service availability

### Push (StatsD, OpenTelemetry)

```python
# Service pushes metrics
statsd.increment('orders.created', 1)
```

Pros:
* Suitable for batch jobs
* Works when aggregator is unavailable
* Less load on the service

Cons:
* Service must know where to send
* Harder to debug

## Retention

### Prometheus

```yaml
# How long to keep metrics
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# Retention 30 days
storage:
  tsdb:
    retention.time: 30d
```

### Storage cost

Basic rule:
```
1 metric with 5 labels = 10 bytes/s
10,000 metrics = 100,000 bytes/s = 8.64 GB/day
```

Cost reduction:
* Reducing cardinality
* Downsampling (store only 1 point/minute)
* Shorter retention for high-cardinality metrics

## Best practices

✅ **Do**
* Start with basic metrics (requests, errors, duration)
* Use labels for filtering
* Name metrics clearly
* Expose response time (histogram)
* Monitor resource usage
* Test metrics
* Document your metrics
* Use consistent units (seconds, bytes)

❌ **Don't**
* Log instead of metrics
* Use high-cardinality labels
* Mix metric and log labels
* Ignore metrics in development
* Create too many metrics
* Use unclear names

## Additional resources

* [Prometheus Best Practices](https://prometheus.io/docs/practices/naming/)
* [OpenTelemetry Metrics](https://opentelemetry.io/docs/concepts/signals/metrics/)
* [RED Method](https://grafana.com/blog/2018/08/02/the-red-method/)
* [USE Method](http://www.brendangregg.com/usemethod.html)
* [Metrics Cardinality Guide](https://www.robustperception.io/cardinality-is-key/)
