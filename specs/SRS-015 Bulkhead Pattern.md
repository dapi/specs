# SRS-015 Bulkhead Pattern

## Definition

Bulkhead Pattern is a resource isolation technique where the system is divided into isolated compartments (bulkheads), so that a failure in one compartment does not affect other parts of the system.

The name comes from ship bulkheads, which prevent the entire ship from flooding when one compartment is breached.

## Why it's needed

Without Bulkhead:
```
All threads → One connection pool (20) → Database
If one thread blocks → All others suffer
```

With Bulkhead:
```
Thread A → Pool 1 (10 connections) → Database
Thread B → Pool 2 (10 connections) → Database
Thread C → Pool 3 (10 connections) → Database
Block in one thread → Others work normally
```

## Types of bulkheads

### 1. Separation by consumers

```python
# Pool for payments (critical)
payment_pool = ConnectionPool(
    name="payments",
    max_connections=20,
    min_connections=5
)

# Pool for reports (non-critical)
reporting_pool = ConnectionPool(
    name="reports",
    max_connections=5,
    min_connections=1
)

# Pool for batch operations
batch_pool = ConnectionPool(
    name="batch",
    max_connections=10,
    min_connections=2
)
```

### 2. Separation by resources

```python
# Separate pools for reads and writes
read_pool = ConnectionPool("read", max_connections=25)
write_pool = ConnectionPool("write", max_connections=10)
```

### 3. Separation by priority

```python
# High priority (user operations)
high_pool = ConnectionPool("high_priority", max_connections=15)

# Low priority (background tasks)
low_pool = ConnectionPool("low_priority", max_connections=5)
```

## Configuration

Via environment variables:
```
BULKHEAD_ENABLED=true
BULKHEAD_POOLS=payments,reports,batch

PAYMENTS_POOL_MAX_SIZE=20
PAYMENTS_POOL_MIN_SIZE=5
PAYMENTS_POOL_TIMEOUT_SEC=5

REPORTS_POOL_MAX_SIZE=5
REPORTS_POOL_MIN_SIZE=1
REPORTS_POOL_TIMEOUT_SEC=30
```

## Implementation in Java (HikariCP)

```java
HikariConfig paymentConfig = new HikariConfig();
paymentConfig.setPoolName("payments");
paymentConfig.setMaximumPoolSize(20);
paymentConfig.setMinimumIdle(5);
paymentConfig.setConnectionTimeout(5000);

HikariConfig reportConfig = new HikariConfig();
reportConfig.setPoolName("reports");
reportConfig.setMaximumPoolSize(5);
reportConfig.setMinimumIdle(1);
reportConfig.setConnectionTimeout(30000);

DataSource paymentDS = new HikariDataSource(paymentConfig);
DataSource reportDS = new HikariDataSource(reportConfig);
```

## Implementation in Go

```go
// Pool for user requests
userPool, err := pgxpool.New(ctx, "pool_max_conns=20")

// Pool for internal tasks
internalPool, err := pgxpool.New(ctx, "pool_max_conns=5")
```

## Metrics

```
bulkhead_pool_size (gauge)
bulkhead_pool_active (gauge)
bulkhead_pool_idle (gauge)
bulkhead_pool_waits (counter)
bulkhead_rejected (counter)
bulkhead_wait_duration (histogram)
```

## Usage scenarios

### System with different request types

```
├───────────┐
 API Gateway
└───────────┘
       │
├─────────┐
  Router
└──┐││──┘
   │  │  │
   │  │  ┛───→ reporting_pool (5 conns)
   │  ┛─────→ batch_pool (10 conns)
   ┛────────→ payment_pool (20 conns)
```

### Microservice with multiple databases

```
├───────┐
 Service
└──┐───┘
   │  │
   │  ┛───→ users_db_pool (10 conns)
   ┛─────→ orders_db_pool (15 conns)
```

## Advantages

✅ **Better resilience**
- Failure in one part does not spread
- Limiting blast radius

✅ **Predictable performance**
- Each component has clear limits
- No mutual influence between components

✅ **Flexible scaling**
- Can configure different limits for different needs
- Granular resource management

✅ **Simplified debugging**
- Clear which component uses resources
- Easy to find problems

## Disadvantages

⚠️ **Complex configuration** - need to configure multiple pools
⚠️ **Suboptimal resource usage** - if one pool is loaded and another is idle
⚠️ **Higher load on database** - more open connections

## Best practices

✅ **Do**
* Separate critical and non-critical operations
* Set different timeouts for different pools
* Monitor each pool separately
* Test under load
* Configure circuit breakers for each pool

❌ **Don't**
* Over-granular splitting (>10 pools)
* Use one pool for everything
* Ignore metrics of individual pools
* Allocate resources equally for critical and non-critical operations regardless of priority

## Alternatives

If Bulkhead is too complex:
* [SRS-007 Circuit Breaker](SRS-007%20Circuit%20Breaker.md)
* [SRS-009 Blocking Timeouts](SRS-009%20Blocking%20Timeouts.md)
* [SRP-003 Load Shedding](SRP-003%20Load%20Shedding.md)
