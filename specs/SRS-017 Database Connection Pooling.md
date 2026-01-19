# SRS-017 Database Connection Pooling

## Definition

Connection Pool is a set of reusable database connections that avoids the overhead of establishing a new connection for each request.

## Why pool is needed

Without pool:
```
Request 1: Establish connection (50ms) → Execute query (5ms) → Close (5ms) = 60ms
Request 2: Establish connection (50ms) → Execute query (5ms) → Close (5ms) = 60ms
Request 3: Establish connection (50ms) → Execute query (5ms) → Close (5ms) = 60ms

Total: 180ms
```

With pool:
```
Pool initialization: Establish 10 connections (500ms)

Request 1: Take from pool (0ms) → Execute (5ms) → Return to pool (0ms) = 5ms
Request 2: Take from pool (0ms) → Execute (5ms) → Return to pool (0ms) = 5ms
Request 3: Take from pool (0ms) → Execute (5ms) → Return to pool (0ms) = 5ms

Total: 500ms (once at startup) + 15ms = 515ms
```

## Pool size

### Minimum size
```
DB_POOL_MIN_SIZE=5  # Connections that are always open
```

Note: For microservices with low traffic, 2-5 can be used.

### Maximum size
```
DB_POOL_MAX_SIZE=20  # Maximum number of connections
```

Calculation formula:
```
max_connections = (CORES * 2) + EFFECTIVE_SPINDLE_COUNT

Where:
- CORES: Number of CPU cores
- EFFECTIVE_SPINDLE_COUNT: For disk systems (usually 1)

Example:
4 cores → (4 * 2) + 1 = 9 connections
16 cores → (16 * 2) + 1 = 33 connections
```

For microservices, 10-30 is usually sufficient.

### Queue size
```
DB_POOL_QUEUE_SIZE=50  # How many requests can wait in queue
```

If queue is full, requests are rejected.

## Timeouts

### Connection Timeout
```
DB_POOL_CONNECTION_TIMEOUT=5000  # milliseconds
```

Time to wait for connection establishment to DB.

### Idle Timeout
```
DB_POOL_IDLE_TIMEOUT=600000  # 10 minutes
```

Time after which unused connections are closed.

### Max Lifetime
```
DB_POOL_MAX_LIFETIME=1800000  # 30 minutes
```

Maximum lifetime of a connection (prevents leaks).

### Acquire Timeout
```
DB_POOL_ACQUIRE_TIMEOUT=5000  # milliseconds
```

Time to wait for a free connection from the pool.

## Framework configuration

### Java (HikariCP - recommended)

```properties
# Connection
spring.datasource.url=jdbc:postgresql://db:5432/mydb
spring.datasource.username=user
spring.datasource.password=pass

# HikariCP
spring.datasource.hikari.minimumIdle=5
spring.datasource.hikari.maximumPoolSize=20
spring.datasource.hikari.connectionTimeout=5000
spring.datasource.hikari.idleTimeout=600000
spring.datasource.hikari.maxLifetime=1800000
spring.datasource.hikari.leakDetectionThreshold=60000

# Connection parameters
spring.datasource.hikari.connectionTestQuery=SELECT 1
spring.datasource.hikari.validationTimeout=3000
```

### Node.js (node-postgres)

```javascript
const pool = new Pool({
  host: 'db',
  port: 5432,
  database: 'mydb',
  user: 'user',
  password: 'pass',

  // Pool settings
  min: 5,                    // minimum pool size
  max: 20,                   // maximum pool size

  // Timeouts
  connectionTimeoutMillis: 5000,
  idleTimeoutMillis: 600000,
  maxLifetimeMillis: 1800000,

  // Test connection
  statement_timeout: 30000,
  query_timeout: 30000,

  // Health check
  keepAlive: true,
  keepAliveInitialDelayMillis: 10000
});
```

### Python (SQLAlchemy)

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import Pool

engine = create_engine(
    "postgresql://user:pass@db:5432/mydb",
    pool_size=5,              # minimum
    max_overflow=15,          # max = pool_size + max_overflow = 20
    pool_timeout=5,           # connection timeout
    pool_recycle=1800,        # max lifetime in seconds
    pool_pre_ping=True,       # test connection before using
    echo_pool=True            # log pool events
)
```

### Ruby (ActiveRecord)

```yaml
# config/database.yml
production:
  adapter: postgresql
  encoding: unicode
  database: mydb
  pool: 20                 # max pool size
  min_messages: log
  prepared_statements: true

  # Timeouts
  timeout: 5000            # connection timeout
  checkout_timeout: 5      # acquire timeout
  reaping_frequency: 10    # how often to check for dead connections
  dead_connection_timeout: 30
```

### Go (database/sql)

```go
db, err := sql.Open("postgres", "postgresql://user:pass@db:5432/mydb")

// Set pool size
db.SetMaxOpenConns(20)        // max pool size
db.SetMaxIdleConns(5)         // min idle connections
db.SetConnMaxLifetime(30 * time.Minute)   // max lifetime
db.SetConnMaxIdleTime(10 * time.Minute)   // idle timeout

// Test connection
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
err = db.PingContext(ctx)
```

## Metrics

```
db_pool_active_connections (gauge)    # active connections
db_pool_idle_connections (gauge)      # idle connections
db_pool_pending_requests (gauge)      # requests in queue
db_pool_acquired_total (counter)      # acquired from pool
db_pool_released_total (counter)      # returned to pool
db_pool_created_total (counter)       # new connections created
db_pool_closed_total (counter)        # closed
db_pool_acquire_duration (histogram)  # acquisition time
db_pool_exceptions_total (counter)    # errors
```

## Health Checks

### Connection Test Query

```sql
-- Lightweight query for connection test
SELECT 1 AS health_check;

-- For PostgreSQL you can use
SELECT pg_postmaster_start_time();

-- For MySQL
SELECT VERSION();
```

### Periodic check (Pool)

If a connection hasn't been used for N seconds, test it before use.

```
DB_POOL_VALIDATION_TIMEOUT=3000  # milliseconds
DB_POOL_VALIDATION_INTERVAL=30000  # every 30 seconds
```

## Problems and solutions

### Connection Leaks

Problem: Connections aren't returned to pool.

Solutions:
* Always close connections (try/finally)
* Use leak detection (HikariCP)
* Set max lifetime

```
HikariCP leakDetectionThreshold=60000  # log if connection > 60 sec
```

### Pool Exhaustion

Problem: All connections are busy, new requests wait.

Solutions:
* Increase max pool size
* Reduce query time
* Use connection timeout
* Implement circuit breaker
* Check slow queries

```
DB_POOL_ACQUIRE_TIMEOUT=5000  # reject after 5 sec waiting
```

### Idle Connections

Problem: Connections idle and database closes them.

Solutions:
* Set idle timeout < database idle timeout
* Use keep alive
* Set min pool size instead of pre-allocating

```
DB_POOL_IDLE_TIMEOUT=600000  # close if idle for 10 minutes
```

### Connection Storm

Problem: Many requests simultaneously create new connections.

Solutions:
* Pre-allocate connections (min pool size)
* Set connection creation rate limit
* Use queue for requests

## Best practices

✅ **Do**
* Always use connection pool
* Configure min and max pool size
* Set timeouts (connection, acquire, idle)
* Enable connection health checks
* Monitor pool metrics
* Test pool under load
* Use prepared statements
* Close connections in finally
* Set max lifetime (prevents leaks)
* Enable leak detection in development

❌ **Don't**
* Create new connections for each request
* Ignore pool errors
* Use unlimited pool size
* Ignore timeout configuration
* Share pool between different databases
* Store state in connection
* Ignore pool metrics

## Default settings

```
DB_POOL_MIN_SIZE=5
DB_POOL_MAX_SIZE=20
DB_POOL_ACQUIRE_TIMEOUT=5000
DB_POOL_CONNECTION_TIMEOUT=5000
DB_POOL_IDLE_TIMEOUT=600000
DB_POOL_MAX_LIFETIME=1800000
DB_POOL_QUEUE_SIZE=50
DB_POOL_VALIDATION_INTERVAL=30000
```

## Configuration in Kubernetes

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Pool settings
  DB_POOL_MIN_SIZE: "5"
  DB_POOL_MAX_SIZE: "20"

  # Timeouts
  DB_POOL_ACQUIRE_TIMEOUT: "5000"
  DB_POOL_CONNECTION_TIMEOUT: "5000"
  DB_POOL_IDLE_TIMEOUT: "600000"
  DB_POOL_MAX_LIFETIME: "1800000"

  # Queue
  DB_POOL_QUEUE_SIZE: "50"
---
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_URL: postgresql://user:pass@db:5432/mydb
```

## Pool size by environment

**Development:**
```
min: 2, max: 5  # Few requests in dev
```

**Staging:**
```
min: 3, max: 10  # Medium load
```

**Production:**
```
min: 5, max: 20  # High load
```

**High Load Production:**
```
min: 10, max: 50  # Very high load
```

## Additional resources

* [HikariCP Configuration](https://github.com/brettwooldridge/HikariCP)
* [PostgreSQL Connection Pooling](https://www.postgresql.org/docs/current/libpq-pgpass.html)
* [MySQL Connection Pooling](https://dev.mysql.com/doc/connector-j/en/connector-j-usagenotes-j2ee-concepts-connection-pooling.html)
* [Common Connection Pool Misconfigurations](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
