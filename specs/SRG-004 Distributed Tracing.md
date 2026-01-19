# SRG-004 Distributed Tracing

## Definition

Distributed Tracing is a method of monitoring requests that pass through multiple services, allowing tracking of the complete path from entry to exit.

## Why it's needed

Without tracing:
```
[Client] → [API Gateway] → [Service A] → [Service B] → [Service C] → [Database]
                                                              ↓
                                                    Client waits 5 seconds
                                                    Where is the problem???
```

With tracing:
```
[Client] → [API Gateway: 10ms] → [Service A: 50ms] → [Service B: 3000ms] → [Service C: 50ms] → [Database: 30ms]
                                                              ↓
                                                    Problem in Service B!
```

## Concepts

### Span

A single operation in a trace. Has:
* Start and end
* Name (operation name)
* Tags (metadata)
* Logs (events)

```
┌──────────────────────────────Span────────────────────────────────┐
│ Name: GET /api/users                                             │
│ Start: 2024-12-12T10:00:00.000Z                                  │
│ End: 2024-12-12T10:00:00.050Z                                    │
│ Duration: 50ms                                                    │
│                                                                   │
│ Tags:                                                             │
│   - http.method = GET                                             │
│   - http.url = /api/users                                         │
│   - http.status_code = 200                                        │
│   - service.name = user-service                                   │
│                                                                   │
│ Logs:                                                             │
│   - [0ms] Starting request                                        │
│   - [10ms] Database query started                                 │
│   - [40ms] Database query completed                               │
│   - [50ms] Request completed                                      │
└───────────────────────────────────────────────────────────────────┘
```

### Trace

A chain of spans showing the complete request path.

```
┌──────────────────────────────────────Trace─────────────────────────────────────┐
│ Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736                                  │
│                                                                                │
│ ┌────────Span 1────────┐                                                       │
│ │ GET /api/users       │                                                       │
│ │ Service: API Gateway │                                                       │
│ │ Duration: 60ms       │                                                       │
│ └──────────┬───────────┘                                                       │
│            │                                                                  │
│    ┌───────▼────────┐      ┌────────Span 3────────┐                          │
│    │ GET /users     │      │  Query users         │                          │
│    │ Service: User  │─────▶│  Service: Database   │                          │
│    │ Duration: 50ms │      │  Duration: 30ms      │                          │
│    └───────┬────────┘      └──────────────────────┘                          │
│            │                                                                  │
│    ┌───────▼────────┐                                                          │
│    │ POST /notify   │                                                          │
│    │ Service: Notif │                                                          │
│    │ Duration: 20ms │                                                          │
│    └────────────────┘                                                          │
└────────────────────────────────────────────────────────────────────────────────┘
```

### Trace Context

Information passed between services:

```http
X-Trace-Id: 4bf92f3577b34da6a3ce929d0e0e4736
X-Span-Id: 00f067aa0ba902b7
X-Parent-Span-Id: 3ce929d0e0e4736a
X-Sampled: 1
```

In W3C Trace Context standard:
```http
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
#     version - trace_id ------------------ span_id -------------- flags
```

## Instrumentation

### Python (OpenTelemetry)

```python
from opentelemetry import trace
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

# Setup tracer
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Configure Jaeger exporter
jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger-agent",
    agent_port=6831,
)

span_processor = BatchSpanProcessor(jaeger_exporter)
trace.get_tracer_provider().add_span_processor(span_processor)

# Instrument app
app = FastAPI()
FastAPIInstrumentor.instrument_app(app)
RequestsInstrumentor().instrument()  # Auto-trace HTTP requests

@app.get("/api/users")
async def get_users():
    with tracer.start_as_current_span("get_users") as span:
        # Add attributes
        span.set_attribute("http.method", "GET")
        span.set_attribute("service.name", "user-service")

        # Add event
        span.add_event("Fetching users from database")

        # Database query
        users = await fetch_users_from_db()

        span.add_event("Users fetched", {"count": len(users)})

        return users

async def fetch_users_from_db():
    with tracer.start_as_current_span("database_query"):
        # Database operation here
        return await db.query("SELECT * FROM users")
```

### Java (Spring Boot + OpenTelemetry)

```java
@SpringBootApplication
public class UserServiceApplication {
    public static void main(String[] args) {
        // Auto instrumentation via Java agent
        // java -javaagent:opentelemetry-javaagent.jar -jar app.jar
        SpringApplication.run(UserServiceApplication.class, args);
    }
}

@RestController
public class UserController {

    @Autowired
    private Tracer tracer;

    @GetMapping("/api/users")
    public List<User> getUsers() {
        Span span = tracer.spanBuilder("get_users")
            .setSpanKind(SpanKind.SERVER)
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            span.setAttribute("http.method", "GET");
            span.setAttribute("service.name", "user-service");

            span.addEvent("Fetching users from database");

            List<User> users = userRepository.findAll();

            span.addEvent("Users fetched");
            Attributes.of(AttributeKey.longKey("count"), (long) users.size())
            );

            return users;

        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, "Failed to fetch users");
            throw e;
        } finally {
            span.end();
        }
    }
}

// In repository
public class UserRepository {
    @Autowired
    private Tracer tracer;

    public List<User> findAll() {
        Span span = tracer.spanBuilder("database_query")
            .setSpanKind(SpanKind.CLIENT)
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            span.setAttribute("db.system", "postgresql");
            span.setAttribute("db.statement", "SELECT * FROM users");

            // Execute query
            return jdbcTemplate.query("SELECT * FROM users", userMapper);
        } finally {
            span.end();
        }
    }
}
```

### Go

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/exporters/jaeger"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.17.0"
)

func initTracer() *sdktrace.TracerProvider {
    exp, err := jaeger.New(jaeger.WithCollectorEndpoint(
        jaeger.WithEndpoint("http://jaeger:14268/api/traces"),
    ))
    if err != nil {
        panic(err)
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exp),
        sdktrace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName("user-service"),
            semconv.ServiceVersion("1.0.0"),
        )),
    )

    otel.SetTracerProvider(tp)
    return tp
}

func getUsers(w http.ResponseWriter, r *http.Request) {
    tracer := otel.Tracer("user-service")

    // Extract context from incoming request
    ctx := r.Context()
    ctx, span := tracer.Start(ctx, "get_users")
    defer span.End()

    span.SetAttributes(
        attribute.String("http.method", r.Method),
        attribute.String("http.url", r.URL.Path),
    )

    span.AddEvent("Fetching users from database")

    users, err := fetchUsers(ctx)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "Failed to fetch users")
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    span.SetAttributes(attribute.Int("user.count", len(users)))

    json.NewEncoder(w).Encode(users)
}

func fetchUsers(ctx context.Context) ([]User, error) {
    tracer := otel.Tracer("database")

    ctx, span := tracer.Start(ctx, "database_query")
    defer span.End()

    span.SetAttributes(
        attribute.String("db.system", "postgresql"),
        attribute.String("db.statement", "SELECT * FROM users"),
    )

    // Execute query with context
    rows, err := db.QueryContext(ctx, "SELECT * FROM users")
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    // Process rows...
    return users, nil
}
```

## Propagation

### HTTP Headers (W3C Trace Context)

```python
import requests
from opentelemetry.propagate import inject

def call_another_service():
    headers = {}
    inject(headers)  # Inject current trace context

    response = requests.get(
        "http://other-service/api",
        headers=headers
    )
```

### Message Queues

```python
from kafka import KafkaProducer
from opentelemetry.propagate import inject

producer = KafkaProducer()

def send_message():
    headers = []
    inject(headers)  # Inject trace context

    producer.send(
        'my-topic',
        value=b'message',
        headers=headers
    )
```

### gRPC

```go
import (
    "google.golang.org/grpc"
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)

// Server
creds := grpc.Creds(credentials.NewTLS(tlsConfig))
server := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
    grpc.Creds(creds),
)

// Client
conn, err := grpc.Dial(
    address,
    grpc.WithTransportCredentials(credentials.NewTLS(tlsConfig)),
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
)
```

## Sampling

### Why sampling is needed

100% tracing:
```
10,000 requests/second × 10 spans = 100,000 spans/second
= 8.64 billion spans/day
```

This is very expensive for storage and analysis.

### Sampling types

#### 1. Probability Sampling

```go
import (
    "go.opentelemetry.io/otel/sdk/trace"
    "go.opentelemetry.io/otel/sdk/trace/tracetest"
)

// 10% of traces
sampler := trace.TraceIDRatioBased(0.10)
tp := trace.NewTracerProvider(
    trace.WithSampler(sampler),
)
```

#### 2. Rate Limiting Sampling

```java
// Max 100 traces per second
Sampler sampler = Sampler.rateLimiting(100);

SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .setSampler(sampler)
    .build();
```

#### 3. Adaptive Sampling

```python
# Kamon (JVM)
kamon {
  trace {
    sampler = adaptive {
      interval = 1 second
      target-traces-per-interval = 100
    }
  }
}
```

### When to collect 100%

* Development/Staging
* Errors (status >= 500)
* Critical endpoints (/payment, /checkout)
* Paid clients (enterprise)

```python
# Always collect error traces
def should_sample(attrs):
    status_code = attrs.get("http.status_code", 0)

    if status_code >= 500:
        return ALWAYS_SAMPLE  # Always collect

    return PROBABILITY_SAMPLE(0.10)  # 10% for successful
```

## Backend for storage

### Jaeger

```yaml
# docker-compose.yml
jaeger:
  image: jaegertracing/all-in-one:latest
  ports:
    - "16686:16686"  # UI
    - "14268:14268"  # HTTP collector
    - "14250:14250"  # gRPC collector
  environment:
    COLLECTOR_ZIPKIN_HTTP_PORT: 9411
```

### Tempo (Grafana)

```yaml
tempo:
  image: grafana/tempo:latest
  command: ["-config.file=/etc/tempo.yaml"]
  volumes:
    - ./tempo.yaml:/etc/tempo.yaml
  ports:
    - "3200:3200"  # Tempo
    - "4317:4317"  # gRPC
    - "4318:4318"  # HTTP
```

```yaml
# tempo.yaml
distributor:
  receivers:
    otlp:
      protocols:
        http:
        grpc:

compactor:
  compaction:
    block_retention: 168h  # 7 days
```

### Elasticsearch

For large volumes:
```yaml
jaeger-collector:
  environment:
    SPAN_STORAGE_TYPE: elasticsearch
    ES_SERVER_URLS: http://elasticsearch:9200
    ES_INDEX_PREFIX: jaeger
```

## Configuration

### OpenTelemetry Collector

```yaml
# otel-collector.yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024

  # Sampling
  probabilistic_sampler:
    sampling_percentage: 10

  # Add attributes
  attributes:
    actions:
      - key: environment
        value: production
        action: insert

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true

  # Also export to Prometheus
  prometheus:
    endpoint: "0.0.0.0:8889"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [probabilistic_sampler, batch, attributes]
      exporters: [jaeger]

    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
```

### Service configuration

```
# Tracing
TRACING_ENABLED=true
TRACING_SERVICE_NAME=user-service
TRACING_SERVICE_VERSION=1.0.0
TRACING_ENVIRONMENT=production

# OTLP Exporter
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
OTEL_EXPORTER_OTLP_INSECURE=true

# Sampling
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.10  # 10%

# Resource attributes
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production,service.namespace=payments
```

## Usage in code

### Adding tags

```python
span.set_attributes({
    "user.id": user_id,
    "user.tier": "premium",
    "order.value": order_value,
    "payment.method": "card",
})
```

### Adding events

```python
span.add_event("Order created", {
    "order.id": order_id,
    "user.id": user_id,
    "value": value,
})

span.add_event("Payment processed", {
    "payment.id": payment_id,
    "status": "success",
})
```

### Recording errors

```python
except Exception as e:
    span.record_exception(e)
    span.set_status(StatusCode.ERROR, str(e))
    raise
```

## W3C Trace Context

Standard for passing context:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
           │  │                                 │               │
           │  │                                 │               └─ Flags (sampled)
           │  │                                 └─ Parent Span ID (16 bytes)
           │  └─ Trace ID (32 bytes)
           └─ Version (2 hex digits)

tracestate: congo=t61rcWkgMzE  # Vendor-specific data
```

## Best practices

✅ **Do**
* Use W3C Trace Context for HTTP
* Aggregate spans in batches before sending
* Add service name and version as resource
* Use sampling to reduce load
* Add errors to spans
* Use semantic conventions (attributes)
* Store traces for at least 7 days
* Collect 100% for errors

❌ **Don't**
* Create spans for every line of code (over-tracing)
* Skip propagation (losing traces)
* Collect 100% traces in production (expensive)
* Ignore timeouts when sending traces
* Add PII (personally identifiable information)
* Create too many span events

## Additional resources

* [OpenTelemetry Tracing](https://opentelemetry.io/docs/concepts/signals/traces/)
* [W3C Trace Context](https://www.w3.org/TR/trace-context/)
* [Jaeger Tracing](https://www.jaegertracing.io/)
* [Grafana Tempo](https://grafana.com/docs/tempo/)
* [Zipkin](https://zipkin.io/)
* [Distributed Systems Observability](https://www.oreilly.com/library/view/distributed-systems-observability/9781492033431/)
