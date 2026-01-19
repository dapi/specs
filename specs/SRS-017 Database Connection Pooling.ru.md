# SRS-017 Database Connection Pooling (Пул соединений с базой данных)

## Определение

Connection Pool - это набор повторно используемых соединений с базой данных, который позволяет избежать накладных расходов на установку нового соединения для каждого запроса.

## Зачем нужен пул

Без пула:
```
Запрос 1: Установить соединение (50ms) → Выполнить запрос (5ms) → Закрыть (5ms) = 60ms
Запрос 2: Установить соединение (50ms) → Выполнить запрос (5ms) → Закрыть (5ms) = 60ms
Запрос 3: Установить соединение (50ms) → Выполнить запрос (5ms) → Закрыть (5ms) = 60ms

Total: 180ms
```

С пулом:
```
Инициализация пула: Установить 10 соединений (500ms)

Запрос 1: Взять из пула (0ms) → Выполнить (5ms) → Вернуть в пул (0ms) = 5ms
Запрос 2: Взять из пула (0ms) → Выполнить (5ms) → Вернуть в пул (0ms) = 5ms
Запрос 3: Взять из пула (0ms) → Выполнить (5ms) → Вернуть в пул (0ms) = 5ms

Total: 500ms (один раз при старте) + 15ms = 515ms
```

## Размер пула

### Минимальный размер
```
DB_POOL_MIN_SIZE=5  # Количество соединений, которые всегда открыты
```

P.S. Для микросервиса с малым трафиком можно использовать 2-5.

### Максимальный размер
```
DB_POOL_MAX_SIZE=20  # Максимальное количество соединений
```

Формула расчета:
```
max_connections = (CORES * 2) + EFFECTIVE_SPINDLE_COUNT

Где:
- CORES: Количество ядер CPU
- EFFECTIVE_SPINDLE_COUNT: Для дисковых систем (обычно 1)

Пример:
4 ядра → (4 * 2) + 1 = 9 соединений
16 ядер → (16 * 2) + 1 = 33 соединения
```

Для микросервисов обычно 10-30 достаточно.

### Размер очереди ожидания
```
DB_POOL_QUEUE_SIZE=50  # Сколько запросов могут ждать в очереди
```

Если очередь переполнена, запросы отклоняются.

## Timeout-ы

### Connection Timeout
```
DB_POOL_CONNECTION_TIMEOUT=5000  # milliseconds
```

Время ожидания установления соединения с БД.

### Idle Timeout
```
DB_POOL_IDLE_TIMEOUT=600000  # 10 minutes
```

Время, после которого неиспользуемое соединение закрывается.

### Max Lifetime
```
DB_POOL_MAX_LIFETIME=1800000  # 30 minutes
```

Максимальное время жизни соединения (предотвращает утечки).

### Acquire Timeout
```
DB_POOL_ACQUIRE_TIMEOUT=5000  # milliseconds
```

Время ожидания свободного соединения из пула.

## Конфигурация по фреймворкам

### Java (HikariCP - рекомендуется)

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

# Параметры соединения
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

## Метрики

```
db_pool_active_connections (gauge)    # активные соединения
db_pool_idle_connections (gauge)      # простаивающие соединения
db_pool_pending_requests (gauge)      # запросы в очереди
db_pool_acquired_total (counter)      # получено из пула
db_pool_released_total (counter)      # возвращено в пул
db_pool_created_total (counter)       # создано новых
db_pool_closed_total (counter)        # закрыто
db_pool_acquire_duration (histogram)  # время получения
db_pool_exceptions_total (counter)    # ошибки
```

## Health Checks

### Connection Test Query

```sql
-- Легкий запрос для проверки соединения
SELECT 1 AS health_check;

-- Для PostgreSQL можно использовать
SELECT pg_postmaster_start_time();

-- Для MySQL
SELECT VERSION();
```

### Периодическая проверка (Pool)

Если соединение не использовалось N секунд, проверить его перед использованием.

```
DB_POOL_VALIDATION_TIMEOUT=3000  # milliseconds
DB_POOL_VALIDATION_INTERVAL=30000  # every 30 seconds
```

## Проблемы и решения

### Connection Leaks

Проблема: Соединения не возвращаются в пул.

Решения:
* Always close connections (try/finally)
* Use leak detection (HikariCP)
* Set max lifetime

```
HikariCP leakDetectionThreshold=60000  # логировать если соединение > 60 сек
```

### Pool Exhaustion

Проблема: Все соединения заняты, новые запросы ждут.

Решения:
* Увеличить max pool size
* Уменьшить query time
* Использовать connection timeout
* Implement circuit breaker
* Check slow queries

```
DB_POOL_ACQUIRE_TIMEOUT=5000  # отказать после 5 сек ожидания
```

### Idle Connections

Проблема: Соединения простаивают и база их закрывает.

Решения:
* Установить idle timeout < database idle timeout
* Use keep alive
* Set min pool size вместо pre-allocating

```
DB_POOL_IDLE_TIMEOUT=600000  # закрывать если простаивает 10 минут
```

### Connection Storm

Проблема: Много запросов одновременно создают новые соединения.

Решения:
* Pre-allocate connections (min pool size)
* Set connection creation rate limit
* Use queue for requests

## Best practices

✅ **Делать**
* Всегда использовать пул соединений
* Настроить min и max pool size
* Устанавливать timeouts (connection, acquire, idle)
* Enable connection health checks
* Monitor pool metrics
* Test pool under load
* Use prepared statements
* Close connections in finally
* Set max lifetime (prevents leaks)
* Enable leak detection в development

❌ **Не делать**
* Создавать новые соединения для каждого запроса
* Игнорировать ошибки пула
* Использовать unlimited pool size
* Игнорировать timeout настройки
* Share pool между разными БД
* Хранить state в connection
* Игнорировать метрики пула

## Настройки по умолчанию

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

## Конфигурация в Kubernetes

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

## Размер пула по окружениям

**Development:**
```
min: 2, max: 5  # Мало запросов в dev
```

**Staging:**
```
min: 3, max: 10  # Средняя нагрузка
```

**Production:**
```
min: 5, max: 20  # Высокая нагрузка
```

**High Load Production:**
```
min: 10, max: 50  # Очень высокая нагрузка
```

## Дополнительные ресурсы

* [HikariCP Configuration](https://github.com/brettwooldridge/HikariCP)
* [PostgreSQL Connection Pooling](https://www.postgresql.org/docs/current/libpq-pgpass.html)
* [MySQL Connection Pooling](https://dev.mysql.com/doc/connector-j/en/connector-j-usagenotes-j2ee-concepts-connection-pooling.html)
* [Common Connection Pool Misconfigurations](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
