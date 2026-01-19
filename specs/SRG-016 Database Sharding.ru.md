# SRG-016 Database Sharding (Шардирование баз данных)

## Определение

Шардирование базы данных - это техника горизонтального масштабирования, которая распределяет данные по нескольким экземплярам базы данных (шардам). Каждый шард содержит подмножество общих данных, позволяя системе обрабатывать большие объемы данных и более высокую пропускную способность, чем одна база данных.

## Когда шардировать

**Рассмотрите шардирование когда:**
- Одна БД не справляется с нагрузкой на запись
- Объем данных превышает емкость одного сервера
- Реплики чтения не могут достаточно масштабировать чтение
- Комплаенс требует локальности данных (гео-шардинг)

**Избегайте шардирования когда:**
- Вертикальное масштабирование все еще возможно
- Реплики чтения решают проблему
- Модель данных имеет много cross-shard связей
- Команде не хватает экспертизы в распределенных системах

## Стратегии шардирования

### Range-Based шардирование

```
Shard Key: user_id
Диапазон: 1-1M → Shard 1, 1M-2M → Shard 2, etc.

┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Shard 1    │  │   Shard 2    │  │   Shard 3    │
│ user_id 1-1M │  │user_id 1M-2M │  │user_id 2M-3M │
└──────────────┘  └──────────────┘  └──────────────┘
```

**Плюсы:**
- Простая реализация
- Range запросы эффективны внутри шарда

**Минусы:**
- Hotspots на новейшем шарде
- Неравномерное распределение данных
- Сложно перебалансировать

### Hash-Based шардирование

```
Shard = hash(shard_key) % num_shards

┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Shard 0    │  │   Shard 1    │  │   Shard 2    │
│hash % 3 == 0 │  │hash % 3 == 1 │  │hash % 3 == 2 │
└──────────────┘  └──────────────┘  └──────────────┘
```

**Плюсы:**
- Равномерное распределение данных
- Нет hotspots

**Минусы:**
- Range запросы затрагивают все шарды
- Решардинг требует миграции данных

### Consistent Hashing

```
Hash Ring:
          0
         /|\
        / | \
    Shard1  Shard2
       \   /
        \ /
       Shard3
```

**Плюсы:**
- Минимальное перемещение данных при добавлении/удалении шардов
- Лучшее распределение нагрузки

**Минусы:**
- Более сложная реализация
- Нужны виртуальные узлы для баланса

### Directory-Based шардирование

```
┌─────────────────┐
│  Lookup Table   │
│  tenant_1 → S1  │
│  tenant_2 → S2  │
│  tenant_3 → S1  │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌───────┐
│Shard 1│ │Shard 2│
└───────┘ └───────┘
```

**Плюсы:**
- Гибкое назначение шардов
- Легкая миграция tenant'ов

**Минусы:**
- Lookup таблица - SPOF
- Дополнительная задержка на lookup

## Выбор Shard Key

### Хорошие Shard Keys

```sql
-- Высокая кардинальность, равномерное распределение
tenant_id      -- Multi-tenant приложения
user_id        -- User-centric приложения
order_id       -- Системы обработки заказов
region_id      -- Географическое распределение
```

### Плохие Shard Keys

```sql
-- Избегать:
created_at     -- Time-series создает hotspots
status         -- Низкая кардинальность
country        -- Неравномерное распределение
is_active      -- Бинарные значения
```

### Составные Shard Keys

```sql
-- Комбинировать для лучшего распределения
shard_key = hash(tenant_id, user_id)
shard_key = hash(region_id, timestamp)
```

## Паттерны реализации

### Шардирование на уровне приложения

```python
# Python пример
import hashlib

class ShardRouter:
    def __init__(self, shard_configs):
        self.shards = shard_configs
        self.num_shards = len(shard_configs)

    def get_shard(self, shard_key):
        hash_value = int(hashlib.md5(str(shard_key).encode()).hexdigest(), 16)
        shard_id = hash_value % self.num_shards
        return self.shards[shard_id]

    def execute_on_shard(self, shard_key, query, params):
        shard = self.get_shard(shard_key)
        connection = get_connection(shard)
        return connection.execute(query, params)

    def execute_on_all_shards(self, query, params):
        results = []
        for shard in self.shards:
            connection = get_connection(shard)
            results.extend(connection.execute(query, params))
        return results

# Использование
router = ShardRouter([
    {'host': 'shard1.db.local', 'port': 5432},
    {'host': 'shard2.db.local', 'port': 5432},
    {'host': 'shard3.db.local', 'port': 5432},
])

# Запись на конкретный шард
router.execute_on_shard(user_id, "INSERT INTO orders ...", params)

# Чтение с конкретного шарда
router.execute_on_shard(user_id, "SELECT * FROM orders WHERE user_id = %s", [user_id])

# Scatter-gather запрос
all_orders = router.execute_on_all_shards("SELECT * FROM orders WHERE status = 'pending'", [])
```

### PostgreSQL с Citus

```sql
-- Установить расширение Citus
CREATE EXTENSION citus;

-- Добавить worker узлы
SELECT citus_add_node('worker1', 5432);
SELECT citus_add_node('worker2', 5432);

-- Создать распределенную таблицу
CREATE TABLE orders (
    order_id bigserial,
    tenant_id int NOT NULL,
    user_id int NOT NULL,
    total decimal(10,2),
    created_at timestamptz DEFAULT now()
);

-- Распределить по tenant_id
SELECT create_distributed_table('orders', 'tenant_id');

-- Co-located таблицы (одинаковый shard key = один шард)
CREATE TABLE order_items (
    item_id bigserial,
    order_id bigint,
    tenant_id int NOT NULL,
    product_id int,
    quantity int
);

SELECT create_distributed_table('order_items', 'tenant_id');

-- Reference таблица (реплицируется на все шарды)
CREATE TABLE products (
    product_id serial PRIMARY KEY,
    name text,
    price decimal(10,2)
);

SELECT create_reference_table('products');
```

### MySQL с Vitess

```yaml
# vschema.json
{
  "sharded": true,
  "vindexes": {
    "hash": {
      "type": "hash"
    }
  },
  "tables": {
    "orders": {
      "column_vindexes": [
        {
          "column": "user_id",
          "name": "hash"
        }
      ]
    }
  }
}
```

## Cross-Shard операции

### Cross-Shard Joins

```sql
-- Избегать: Cross-shard join (дорого)
SELECT o.*, u.name
FROM orders o
JOIN users u ON o.user_id = u.id;

-- Лучше: Co-locate связанные данные
-- orders и order_items на одном шарде по tenant_id
SELECT o.*, oi.*
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.tenant_id = 123;  -- Один шард

-- Альтернатива: Join на стороне приложения
users = query_users_shard(user_ids)
orders = query_orders_shard(user_ids)
result = join_in_application(users, orders)
```

### Cross-Shard транзакции

```python
# Паттерн two-phase commit
def transfer_between_shards(from_user, to_user, amount):
    shard_from = router.get_shard(from_user)
    shard_to = router.get_shard(to_user)

    # Фаза 1: Prepare
    try:
        shard_from.execute("BEGIN")
        shard_from.execute("UPDATE accounts SET balance = balance - %s WHERE user_id = %s", [amount, from_user])

        shard_to.execute("BEGIN")
        shard_to.execute("UPDATE accounts SET balance = balance + %s WHERE user_id = %s", [amount, to_user])

        # Фаза 2: Commit
        shard_from.execute("COMMIT")
        shard_to.execute("COMMIT")
    except Exception as e:
        shard_from.execute("ROLLBACK")
        shard_to.execute("ROLLBACK")
        raise e
```

### Глобальные последовательности

```sql
-- Централизованный сервер последовательностей
CREATE SEQUENCE global_order_id;
SELECT nextval('global_order_id');

-- UUID подход (без координации)
SELECT gen_random_uuid() AS order_id;

-- Snowflake ID (timestamp + machine + sequence)
-- 41 бит timestamp | 10 бит machine | 12 бит sequence
```

## Решардинг

### Добавление шардов

```python
# Удвоение шардов (hash-based)
# Старый: shard = hash % 2
# Новый: shard = hash % 4

def migrate_to_new_shards():
    for old_shard in [0, 1]:
        for row in get_all_rows(old_shard):
            new_shard = hash(row.shard_key) % 4
            if new_shard != old_shard:
                insert_to_shard(new_shard, row)
                delete_from_shard(old_shard, row.id)
```

### Миграция с Consistent Hashing

```python
# Минимальное перемещение данных с consistent hashing
from uhashring import HashRing

# Добавить новый узел
ring = HashRing(nodes=['shard1', 'shard2', 'shard3'])
ring.add_node('shard4')

# Только ключи, которые мапятся на shard4, нужно мигрировать
for key in all_keys:
    if ring.get_node(key) == 'shard4':
        migrate_key(key, 'shard4')
```

## Мониторинг

```
# Метрики шардов
db_shard_size_bytes{shard="1"} (gauge)
db_shard_rows_count{shard="1"} (gauge)
db_shard_qps{shard="1"} (gauge)
db_cross_shard_queries_total (counter)
db_shard_rebalance_duration_seconds (histogram)

# Пороги алертинга
WARNING: shard_size_imbalance > 20%
WARNING: cross_shard_queries_ratio > 10%
CRITICAL: shard_unavailable
```

## Best practices

**Делать:**
- Тщательно выбирать shard key (высокая кардинальность, равномерное распределение)
- Co-locate связанные данные на одном шарде
- Минимизировать cross-shard операции
- Планировать решардинг с первого дня
- Использовать reference tables для справочных данных
- Непрерывно мониторить баланс шардов
- Тестировать с production-подобными объемами данных

**Не делать:**
- Шардировать преждевременно
- Использовать shard keys с низкой кардинальностью
- Игнорировать стоимость cross-shard запросов
- Предполагать равномерное распределение без проверки
- Забывать о глобальных последовательностях
- Пропускать тестирование backup/recovery для каждого шарда

## Переменные окружения

```bash
# Конфигурация шардирования
DB_SHARD_COUNT=4
DB_SHARD_STRATEGY=hash  # hash|range|directory
DB_SHARD_KEY=tenant_id

# Соединения шардов
DB_SHARD_0_HOST=shard0.db.local
DB_SHARD_0_PORT=5432
DB_SHARD_1_HOST=shard1.db.local
DB_SHARD_1_PORT=5432

# Cross-shard настройки
DB_CROSS_SHARD_TIMEOUT_MS=30000
DB_SCATTER_GATHER_PARALLEL=true
```

## Дополнительные ресурсы

- [Citus Documentation](https://docs.citusdata.com/)
- [Vitess Documentation](https://vitess.io/docs/)
- [Consistent Hashing](https://en.wikipedia.org/wiki/Consistent_hashing)
- [Database Sharding Patterns](https://www.citusdata.com/blog/2018/01/10/sharding-in-plain-english/)
