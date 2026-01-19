# SRP-003 Load Shedding (Отсечение нагрузки)

## Определение

Load Shedding - это механизм активного отказа от обработки части запросов, когда сервис перегружен и не может обработать все входящие запросы.

## Когда использовать

Отсечение нагрузки необходимо, когда:
* Сервис достигает лимита ресурсов (CPU, Memory, Connections)
* Внешние зависимости недоступны или медленно отвечают
* Система находится в состоянии восстановления после сбоя
* Количество запросов превышает обработную способность

## Алгоритмы отсечения

### 1. Фиксированный лимит
```
MAX_CONCURRENT_REQUESTS=100
Отказывать в обслуживании, если активных запросов > 100
```

### 2. Динамический лимит (адаптивное отсечение)
```
Измерять latency и throughput
Уменьшать лимит при увеличении latency
Увеличивать лимит при улучшении показателей
```

### 3. Приоритет запросов
```
HIGH PRIORITY: Платные пользователи, критичные операции
MEDIUM PRIORITY: Авторизованные пользователи
LOW PRIORITY: Гостевые запросы, batch операции
```

### 4. Circuit Breaker + Load Shedding
Проверять состояние Circuit Breaker для внешних зависимостей:
```
IF external_service.circuit_breaker.state == OPEN:
    shed_requests_dependent_on_that_service()
```

## HTTP ответы

При отсечении нагрузки возвращать:
* **503 Service Unavailable** - сервис перегружен
* **429 Too Many Requests** - превышен лимит запросов
* Retry-After заголовок с рекомендуемым временем повторной попытки

## Метрики

Отслеживать и экспонировать метрики:
```
lload_shedding_requests_total (counter)
lload_shedding_requests_rejected (counter)
current_active_requests (gauge)
request_queue_length (gauge)
resource_utilization (gauge)
```

## Реализация

### Конфигурация через переменные окружения
```
LOAD_SHEDDING_ENABLED=true
MAX_CONCURRENT_REQUESTS=1000
REQUEST_QUEUE_LIMIT=5000
HIGH_PRIORITY_QUOTA=0.5  # 50% для высокоприоритетных
```

### Пример на псевдокоде
```python
def handle_request(request):
    # Проверяем лимит
    if active_requests.count() >= MAX_CONCURRENT_REQUESTS:
        if request.priority == 'high':
            # Проверяем квоту высокоприоритетных
            if high_priority_count < HIGH_PRIORITY_QUOTA * MAX_CONCURRENT_REQUESTS:
                return process(request)

        # Отсечение нагрузки
        metrics.shed_request(request)
        return ErrorResponse(503, "Service temporarily overloaded")

    return process(request)
```

## Graceful Degradation

При отсечении нагрузки использовать запасную логику:
* Возвращать кэшированные ответы
* Возвращать упрощенные ответы
* Отключать некритичные функции
* См. [SRS-014 Fallback](SRS-014%20Fallback.ru.md)

## Мониторинг и Alerting

Алерты должны срабатывать при:
* Load shedding активен > 5 минут
* Отсечение > 10% запросов
* Повторяющиеся события отсечения

## Best practices

✅ **Делать**
* Отсекать запросы рано (на входе)
* Возвращать понятные ошибки
* Логировать отсеченные запросы (на debug уровне)
* Предоставлять статистику через метрики
* Тестировать при нагрузочном тестировании

❌ **Не делать**
* Отсечать критичные запросы без приоритизации
* Затягивать запросы в очередь при отсечении
* Игнорировать метрики отсечения
* Включать отсечение по умолчанию без настройки

## Дополнительные ресурсы

* [Netflix: Performance Under Load](https://netflixtechblog.com/performance-under-load-3e6fa9d2b512)
* [Google SRE: Handling Overload](https://sre.google/sre-book/handling-overload/)
* [AWS: Load Shedding Pattern](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/load-shedding.html)
