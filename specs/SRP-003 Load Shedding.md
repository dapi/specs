# SRP-003 Load Shedding

## Definition

Load Shedding is a mechanism of actively refusing to process part of the requests when the service is overloaded and cannot handle all incoming requests.

## When to use

Load shedding is necessary when:
* Service reaches resource limits (CPU, Memory, Connections)
* External dependencies are unavailable or respond slowly
* System is in recovery state after failure
* Number of requests exceeds processing capacity

## Shedding algorithms

### 1. Fixed limit
```
MAX_CONCURRENT_REQUESTS=100
Refuse service if active requests > 100
```

### 2. Dynamic limit (adaptive shedding)
```
Measure latency and throughput
Decrease limit when latency increases
Increase limit when metrics improve
```

### 3. Request priority
```
HIGH PRIORITY: Paid users, critical operations
MEDIUM PRIORITY: Authorized users
LOW PRIORITY: Guest requests, batch operations
```

### 4. Circuit Breaker + Load Shedding
Check Circuit Breaker state for external dependencies:
```
IF external_service.circuit_breaker.state == OPEN:
    shed_requests_dependent_on_that_service()
```

## HTTP responses

When shedding load, return:
* **503 Service Unavailable** - service is overloaded
* **429 Too Many Requests** - request limit exceeded
* Retry-After header with recommended retry time

## Metrics

Track and expose metrics:
```
load_shedding_requests_total (counter)
load_shedding_requests_rejected (counter)
current_active_requests (gauge)
request_queue_length (gauge)
resource_utilization (gauge)
```

## Implementation

### Configuration via environment variables
```
LOAD_SHEDDING_ENABLED=true
MAX_CONCURRENT_REQUESTS=1000
REQUEST_QUEUE_LIMIT=5000
HIGH_PRIORITY_QUOTA=0.5  # 50% for high-priority
```

### Example in pseudocode
```python
def handle_request(request):
    # Check limit
    if active_requests.count() >= MAX_CONCURRENT_REQUESTS:
        if request.priority == 'high':
            # Check high-priority quota
            if high_priority_count < HIGH_PRIORITY_QUOTA * MAX_CONCURRENT_REQUESTS:
                return process(request)

        # Load shedding
        metrics.shed_request(request)
        return ErrorResponse(503, "Service temporarily overloaded")

    return process(request)
```

## Graceful Degradation

When shedding load, use fallback logic:
* Return cached responses
* Return simplified responses
* Disable non-critical features
* See [SRS-014 Fallback](SRS-014%20Fallback.md)

## Monitoring and Alerting

Alerts should fire when:
* Load shedding is active > 5 minutes
* Shedding > 10% of requests
* Repeating shedding events

## Best practices

✅ **Do**
* Shed requests early (at the entrance)
* Return clear error messages
* Log shed requests (at debug level)
* Provide statistics via metrics
* Test during load testing

❌ **Don't**
* Shed critical requests without prioritization
* Queue requests during shedding
* Ignore shedding metrics
* Enable shedding by default without configuration

## Additional resources

* [Netflix: Performance Under Load](https://netflixtechblog.com/performance-under-load-3e6fa9d2b512)
* [Google SRE: Handling Overload](https://sre.google/sre-book/handling-overload/)
* [AWS: Load Shedding Pattern](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/load-shedding.html)
