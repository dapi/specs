# SRS-015 Bulkhead Pattern (Перегородки)

## Определение

Bulkhead Pattern - техника изоляции ресурсов, при которой система разделяется на изолированные секции (перегородки), чтобы сбой в одной секции не повлиял на остальные части системы.

Название происходит от судовых перегородок (bulkhead), которые предотвращают затопление всего корабля при пробитии одного отсека.

## Зачем нужен

Без Bulkhead:
```
Все потоки → Один пул соединений (20) → База данных
Если один поток блокируется → Все остальные страдают
```

С Bulkhead:
```
Поток A → Пул 1 (10 соединений) → База
Поток B → Пул 2 (10 соединений) → База
Поток C → Пул 3 (10 соединений) → База
Блокировка в одном потоке → Остальные работают нормально
```

## Типы перегородок

### 1. Разделение по потребителям

```python
# Пул для платежей (критичный)
payment_pool = ConnectionPool(
    name="payments",
    max_connections=20,
    min_connections=5
)

# Пул для отчетов (некритичный)
reporting_pool = ConnectionPool(
    name="reports",
    max_connections=5,
    min_connections=1
)

# Пул для batch-операций
batch_pool = ConnectionPool(
    name="batch",
    max_connections=10,
    min_connections=2
)
```

### 2. Разделение по ресурсам

```python
# Отдельные пулы для чтения и записи
read_pool = ConnectionPool("read", max_connections=25)
write_pool = ConnectionPool("write", max_connections=10)
```

### 3. Разделение по приоритету

```python
# Высокий приоритет (пользовательские операции)
high_pool = ConnectionPool("high_priority", max_connections=15)

# Низкий приоритет (фоновые задачи)
low_pool = ConnectionPool("low_priority", max_connections=5)
```

## Конфигурация

Через переменные окружения:
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

## Реализация на Java (HikariCP)

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

## Реализация на Go

```go
// Пул для пользовательских запросов
userPool, err := pgxpool.New(ctx, "pool_max_conns=20")

// Пул для внутренних задач
internalPool, err := pgxpool.New(ctx, "pool_max_conns=5")
```

## Метрики

```
bulkhead_pool_size (gauge)
bulkhead_pool_active (gauge)
bulkhead_pool_idle (gauge)
bulkhead_pool_waits (counter)
bulkhead_rejected (counter)
bulkhead_wait_duration (histogram)
```

## Сценарии применения

### Система с разными типами запросов

```
┌─────────────┐
│ API Gateway │
└──────┬──────┘
       │
┌──────▼──────┐
│  Роутер     │
└──┬──┬──┬───┘
   │  │  │
   │  │  └────→ reporting_pool (5 conns)
   │  └───────→ batch_pool (10 conns)
   └──────────→ payment_pool (20 conns)
```

### Микросервис с несколькими БД

```
┌──────────┐
│ Service  │
└──┬──┬───┘
   │  │
   │  └────→ users_db_pool (10 conns)
   └───────→ orders_db_pool (15 conns)
```

## Преимущества

✅ **Лучшая устойчивость**
- Сбой в одной части не распространяется
- Ограничение blast radius

✅ **Предсказуемая производительность**
- Каждый компонент имеет четкие лимиты
- Нет взаимного влияния между компонентами

✅ **Гибкое масштабирование**
- Можно настроить разные лимиты под разные нужды
- Гранулярное управление ресурсами

✅ **Упрощенное дебажение**
- Понятно, какой компонент использует ресурсы
- Легко найти проблему

## Недостатки

⚠️ **Сложность конфигурации** - нужно настроить несколько пулов
⚠️ **Неоптимальное использование ресурсов** - если один пул загружен, а другой простаивает
⚠️ **Большая нагрузка на базу** - больше открытых соединений

## Best practices

✅ **Делать**
* Разделять критичные и некритичные операции
* Устанавливать разные timeouts для разных пулов
* Мониторить каждый пул отдельно
* Тестировать при нагрузке
* Настраивать circuit breakers для каждого пула

❌ **Не делать**
* Слишком мелкое дробление (>10 пулов)
* Использовать один пул для всего
* Игнорировать метрики отдельных пулов
* Без разницы выделять ресурсы под критичные и некритичные операции

## Альтернативы

Если Bulkhead слишком сложен:
* [SRS-007 Circuit Breaker](SRS-007%20Circuit%20Breaker.ru.md)
* [SRS-009 Blocking Timeouts](SRS-009%20Blocking%20Timeouts.ru.md)
* [SRP-003 Load Shedding](SRP-003%20Load%20Shedding.ru.md)
