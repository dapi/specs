# SRG-009 Database Migrations (Миграции баз данных в production)

Database Migrations - это процесс управления схемой и данными базы данных путем применения упорядоченных изменений, который позволяет развивать структуру базы данных вместе с приложением, сохраняя целостность данных в production окружении.

---

## Антипаттерны миграций ❌

### ❌ Ручные изменения схемы
```bash
# НЕ ДЕЛАТЬ!
$ psql production_db
> ALTER TABLE users ADD COLUMN email VARCHAR(255);  # Без резервной копии
> DELETE FROM logs WHERE created_at < '2024-01-01';  # Без WHERE проверки
```
**Риски:**
- Ошибки в production
- Невозможность отката
- Потеря данных
- Несогласованность между окружениями

### ❌ Без миграций (изменения вручную)
```sql
-- База в каждом окружении - ручная работа
-- Невозможно отследить версии
-- Невозможно откатить
```

### ❌ Без резервных копий
```bash
# Единственная копия базы - это production
# Нет тестирования restore процедуры
```

### ❌ Выполнение миграций без downtime планирования
```bash
# Запуск миграции в peak hours
# Нет плана B
```

---

## Принципы управления миграциями ✅

### 1. Версионирование схемы

```bash
# Структура репозитория
db/
├── migrations/
│   ├── 001_create_users.sql
│   ├── 002_create_orders.sql
│   ├── 003_add_user_email.sql
│   └── 004_add_order_status.sql
├── seeds/
│   ├── development.sql
│   └── test.sql
├── rollback/
│   └── 003_add_user_email_rollback.sql
└── schema.rb  # или schema.sql
```

### 2. Idempotent migrations

```sql
-- 003_add_user_email.sql
-- Идемпотентная миграция - можно запускать несколько раз

-- Проверяем, не существует ли колонка
DO $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.columns
        WHERE table_name = 'users' AND column_name = 'email'
    ) THEN
        ALTER TABLE users ADD COLUMN email VARCHAR(255);
        CREATE INDEX idx_users_email ON users(email);
    END IF;
END $$;
```

### 3. Резервные копии (Always backup first!)

```bash
#!/bin/bash
# backup-and-migrate.sh

# Создаем резервную копию
pg_dump -h $DB_HOST -U $DB_USER -d $DB_NAME -F c -f backup-$(date +%Y%m%d_%H%M%S).bak

# Валидация бэкапа
if ! pg_restore --list backup-*.bak | head -10; then
    echo "❌ Backup validation failed!"
    exit 1
fi

echo "✅ Backup created and validated"
```

---

## Виды миграций

### Type 1: Expand/Contract Pattern (Zero-downtime migrations)

```sql
-- Шаг 1: EXPAND - Добавляем новую колонку
-- Миграция: 005_add_new_column.sql
-- Downtime: None
-- Backward compatible: Yes

START TRANSACTION;

ALTER TABLE users
ADD COLUMN email_verified BOOLEAN DEFAULT false;

-- Создаем триггер для синхронизации данных
CREATE OR REPLACE FUNCTION sync_email_verified()
RETURNS TRIGGER AS $$
BEGIN
    NEW.email_verified = (NEW.verified_at IS NOT NULL);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_sync_verified
    BEFORE INSERT OR UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION sync_email_verified();

COMMIT;
```

```sql
-- Шаг 2: DUAL WRITE - Пишем в обе колонки
-- Приложение обновлено для записи в обе колонки
-- Старые версии приложения все еще работают

-- Приложение v1: Записывает в verified_at
-- Приложение v2: Записывает и в verified_at, и в email_verified

-- Миграция данных (в фоне)
UPDATE users
SET email_verified = (verified_at IS NOT NULL)
WHERE email_verified IS NULL;
```

```sql
-- Шаг 3: CONTRACT - Удаляем старую колонку
-- Миграция: 006_remove_old_column.sql
-- Downtime: None (после того как все инстансы обновлены)
-- Backward compatible: No (после деплоя)

-- Сначала отключаем триггер
DROP TRIGGER users_sync_verified ON users;
DROP FUNCTION sync_email_verified();

-- Удаляем старую колонку
-- ALTER TABLE users DROP COLUMN verified_at;
```

### Type 2: Add-only Migrations (Safe)

```sql
-- Только добавляем, не удаляем
-- Всегда можно откатить

-- Добавляем колонку (может быть NULL или имеет DEFAULT)
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Создаем новую таблицу
CREATE TABLE user_profiles (...);

-- Добавляем новый индекс
CREATE INDEX CONCURRENTLY idx_users_phone ON users(phone);
-- CONCURRENTLY = без блокировки таблицы
```

### Type 3: Heavy Migrations (Split and Conquer)

```python
#!/usr/bin/env python3
# large_migration.py

# Для больших таблиц (миллионы записей)
# Мигрируем в батчах для минимизации блокировок

import psycopg2
import time

BATCH_SIZE = 10000
SLEEP_BETWEEN_BATCHES = 0.5  # seconds

def migrate_large_table():
    conn = psycopg2.connect(os.getenv('DATABASE_URL'))
    cursor = conn.cursor()

    # Мигрируем в батчах
    while True:
        cursor.execute(f"""
            UPDATE users
            SET new_column = calculate_value(old_column)
            WHERE new_column IS NULL
            LIMIT {BATCH_SIZE}
        """)

        updated = cursor.rowcount
        conn.commit()

        print(f"Updated {updated} rows")

        if updated == 0:
            break

        time.sleep(SLEEP_BETWEEN_BATCHES)

    cursor.close()
    conn.close()

if __name__ == '__main__':
    migrate_large_table()
```

### Type 4: Backfill Migrations

```sql
-- Заполняем новую колонку данными из существующих

-- Подход 1: Online backfill (в фоне)
-- Запускаем после добавления колонки

CREATE OR REPLACE FUNCTION backfill_users_name()
RETURNS void AS $$
DECLARE
    batch_size INT := 1000;
    last_id INT := 0;
    total_updated INT := 0;
BEGIN
    LOOP
        UPDATE users
        SET full_name = CONCAT(first_name, ' ', last_name)
        WHERE id > last_id
          AND full_name IS NULL
        ORDER BY id
        LIMIT batch_size;

        IF FOUND THEN
            total_updated := total_updated + batch_size;
            last_id := (SELECT max(id) FROM users WHERE full_name IS NOT NULL);
            COMMIT;
            PERFORM pg_sleep(0.1);  -- Небольшая пауза
        ELSE
            EXIT;
        END IF;
    END LOOP;

    RAISE NOTICE 'Updated % rows', total_updated;
END;
$$ LANGUAGE plpgsql;

-- Запускаем в фоновом процессе
SELECT backfill_users_name();
```

---

## Фреймворки миграций по языкам

### Ruby on Rails (Active Record)

```ruby
# db/migrate/20240115000001_add_email_to_users.rb
class AddEmailToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :email, :string
    add_index :users, :email, unique: true
  end

  def down
    remove_index :users, :email
    remove_column :users, :email
  end
end

# Выполнение
$ rails db:migrate
$ rails db:rollback  # Откат последней миграции
$ rails db:migrate:down VERSION=20240115000001  # Откат конкретной
```

### Django

```python
# migrations/0012_add_email_to_users.py
from django.db import migrations, models

class Migration(migrations.Migration):
    dependencies = [
        ('users', '0011_auto_20240101'),
    ]

    operations = [
        migrations.AddField(
            model_name='user',
            name='email',
            field=models.EmailField(max_length=254, null=True),
        ),
        migrations.AddIndex(
            model_name='user',
            index=models.Index(fields=['email'], name='users_email_idx'),
        ),
    ]

# Выполнение
$ python manage.py migrate
$ python manage.py migrate users 0011  # Откат
```

### Node.js (Sequelize)

```javascript
// migrations/20240115000001-add-email-to-users.js
module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.addColumn('users', 'email', {
      type: Sequelize.STRING,
      allowNull: true,
    });
    await queryInterface.addIndex('users', ['email']);
  },

  down: async (queryInterface, Sequelize) => {
    await queryInterface.removeIndex('users', ['email']);
    await queryInterface.removeColumn('users', 'email');
  }
};

// Выполнение
$ npx sequelize-cli db:migrate
$ npx sequelize-cli db:migrate:undo  # Откат последней
```

### Elixir (Ecto)

```elixir
# priv/repo/migrations/20240115000001_add_email_to_users.exs
defmodule MyApp.Repo.Migrations.AddEmailToUsers do
  use Ecto.Migration

  def change do
    alter table(:users) do
      add :email, :string
    end

    create unique_index(:users, [:email])
  end

  def down do
    drop unique_index(:users, [:email])
    alter table(:users) do
      remove :email
    end
  end
end
```

### Go (golang-migrate)

```bash
# Создание миграции
$ migrate create -ext sql -dir db/migrations add_email_to_users

# Содержимое файлов:
# db/migrations/001_add_email_to_users.up.sql
ALTER TABLE users ADD COLUMN email VARCHAR(255);
CREATE INDEX idx_users_email ON users(email);

# db/migrations/001_add_email_to_users.down.sql
DROP INDEX idx_users_email;
ALTER TABLE users DROP COLUMN email;

# Выполнение
$ migrate -database $DATABASE_URL -path db/migrations up
$ migrate -database $DATABASE_URL -path db/migrations down 1
```

---

## Стратегии деплоя миграций

### Strategy 1: Deploy → Migrate

```bash
# Подход: Сначала деплоим код, потом миграцию
# Риски: Приложение может не запуститься
# Когда: Для обратно совместимых миграций

# Step 1: Deploy new code
$ kubectl rollout restart deployment/app --image=new-version

# Step 2: Run migrations
$ kubectl run migration --rm -i --tty --image=app-migration \
  -- ./migrate up
```

### Strategy 2: Migrate → Deploy

```bash
# Подход: Сначала миграция, потом код
# Безопаснее, но нужно planning

# Step 1: Run migrations
$ ./migrate up

# Step 2: Verify migrations
$ ./migrate verify

# Step 3: Deploy code (после успешной миграции)
$ kubectl rollout restart deployment/app --image=new-version
```

### Strategy 3: Rolling Migrations (для zero-downtime)

```yaml
# deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  template:
    spec:
      initContainers:
        # Миграции выполняются перед стартом pod'а
        - name: db-migration
          image: app:latest
          command: ['npm', 'run', 'db:migrate']
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url

      containers:
        - name: app
          image: app:latest
          ports:
            - containerPort: 3000
```

---

## Unsafe операции и безопасные альтернативы

### ❌ Unsafe: ALTER TABLE на больших таблицах
```sql
ALTER TABLE million_rows_table ADD COLUMN new_col VARCHAR(255);
-- Блокирует таблицу на минуты/часы
-- Записи не могут быть вставлены/обновлены/удалены
```

### ✅ Safe: Using pg_repack or logical replication
```bash
# Устанавливаем pg_repack
$ pg_repack -d mydb -t users \
  --alter 'ADD COLUMN new_col VARCHAR(255)'

# Конкурентно пересоздает таблицу
# Без блокировок
# Можно отменить
```

### ❌ Unsafe: DROP COLUMN
```sql
ALTER TABLE users DROP COLUMN old_column;
-- Потеря данных навсегда
-- Сложно откатить
-- Требует много I/O
```

### ✅ Safe: Mark as deprecated first
```sql
-- Step 1: Mark column as deprecated
COMMENT ON COLUMN users.old_column IS 'DEPRECATED - Remove after 2024/Q2';

-- Step 2: Set column to NULL to освободить space
UPDATE users SET old_column = NULL WHERE old_column IS NOT NULL;

-- Step 3: Remove after verifying no code uses it
-- ALTER TABLE users DROP COLUMN old_column;
```

### ❌ Unsafe: UPDATE without WHERE
```sql
UPDATE users SET status = 'active';
-- Обновляет все записи
-- Лучше добавить WHERE существующих фильтров
```

### ✅ Safe: Batch updates
```sql
-- Мигрируем в батчах
UPDATE users
SET status = 'active'
WHERE id BETWEEN 1 AND 10000
  AND status IS NULL;

-- Проверяем количество
SELECT COUNT(*) FROM users WHERE status IS NULL;

-- Повторяем до тех пор, пока не будет 0
```

---

## Мониторинг миграций

```python
class MigrationMonitoring:
    def monitor_migration(self, migration_id):
        """Мониторим запущенную миграцию"""

        # Проверяем длительность и блокировки
        locks = self.db.query("""
            SELECT * FROM pg_locks
            WHERE NOT granted
              AND relation IN (
                SELECT oid FROM pg_class WHERE relname = 'users'
              )
        """)

        if locks:
            self.send_alert('Migration causing locks', severity='warning')

        # Проверяем deadlocks
        deadlocks = self.db.query("""
            SELECT * FROM pg_stat_database
            WHERE deadlocks > 0
        """)

        if deadlocks:
            self.send_alert('Deadlocks detected during migration', severity='critical')

        # Отмена миграции, если слишком долго
        duration = self.get_migration_duration(migration_id)
        if duration > timedelta(minutes=30):
            self.cancel_migration(migration_id)
            self.send_alert('Migration cancelled due to timeout')
```

---

## Резервное копирование перед миграциями

### Автоматический бэкап

```bash
#!/bin/bash
# backup-db.sh

set -e

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DB_NAME="${DB_NAME:-myapp_production}"
BACKUP_BUCKET="${BACKUP_BUCKET:-myapp-backups}"

echo "🔒 Creating backup of $DB_NAME..."

# Создаем бэкап
pg_dump -h $DB_HOST -U $DB_USER -d $DB_NAME \
  --verbose \
  --no-password \
  --clean \
  --if-exists \
  --no-owner \
  --no-privileges \
  --no-acl \
  -F custom \
  > /tmp/db_backup_$TIMESTAMP.dump

echo "✅ Backup created: /tmp/db_backup_$TIMESTAMP.dump"
echo "📊 Size: $(du -h /tmp/db_backup_$TIMESTAMP.dump | cut -f1)"

# Валидация бэкапа
echo "🔍 Validating backup..."
pg_restore --list /tmp/db_backup_$TIMESTAMP.dump | head -20
echo "✅ Backup validated"

# Сжимаем
gzip /tmp/db_backup_$TIMESTAMP.dump
echo "✅ Backup compressed"

# Загружаем в S3
aws s3 cp /tmp/db_backup_$TIMESTAMP.dump.gz s3://$BACKUP_BUCKET/backups/
echo "✅ Backup uploaded to S3"

# Создаем запись в базе
echo "📝 Logging backup..."
psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c \
  "INSERT INTO db_backups (file_name, created_at, size_bytes) VALUES
   ('db_backup_$TIMESTAMP.dump.gz', NOW(), $(stat -c%s /tmp/db_backup_$TIMESTAMP.dump.gz));"

# Очищаем старые бэкапы (оставляем последние 7 дней)
find /tmp -name "db_backup_*.dump.gz" -mtime +7 -delete

# Перемещаем в архив в S3
aws s3 mv s3://$BACKUP_BUCKET/backups/db_backup_$TIMESTAMP.dump.gz \
       s3://$BACKUP_BUCKET/archive/

echo "🎉 Backup process complete!"
```

---

## Тестирование отката

```bash
#!/bin/bash
# test-rollback.sh

# 1. Создаем тестовую базу
psql -c "CREATE DATABASE test_restore;"

# 2. Восстанавливаем бэкап
pg_restore -d test_restore /tmp/db_backup_test.dump

# 3. Запускаем тесты приложения
./run-tests.sh --database=test_restore

# 4. Откатываем миграцию
./migrate down

# 5. Проверяем, что приложение работает
./run-tests.sh --database=test_restore

# 6. Очищаем
dropdb test_restore

echo "✅ Rollback test passed!"
```

---

## Конфигурация через переменные окружения

```bash
# .env.migrations
MIGRATION_STRATEGY=expand_contract
BACKUP_BEFORE_MIGRATE=true
MIGRATION_TIMEOUT=1800  # 30 minutes
MAX_LOCK_WAIT=60
ALLOW_DDL_IN_PRODUCTION=true

# Database connection
DB_URL=postgres://user:pass@host:5432/dbname
DB_POOL_SIZE=50
DB_STATEMENT_TIMEOUT=30000  # 30 seconds
```

---

## Best practices ✅

### Хранение миграций
- ✅ Версионировать в Git
- ✅ Явные названия (YYYYMMDD_description.sql)
- ✅ Хранить up/down пары
- ✅ Минимизировать ручные правки

### Выполнение
- ✅ Тестировать на dev/staging
- ✅ Запускать в off-peak hours
- ✅ Выполнять upgrade с downgrade планом
- ✅ Отслеживать длительность
- ✅ Выполнять в транзакциях (когда возможно)

### Наблюдаемость
- ✅ Мониторить locks
- ✅ Мониторить I/O
- ✅ Следить за disk space
- ✅ Логировать ошибки
- ✅ Alert на длительные миграции

### Безопасность
- ✅ Резервные копии перед каждой миграцией
- ✅ Иметь откатный план
- ✅ Не мигрировать напрямую в production
- ✅ Иметь emergency procedures

---

*Database Migrations - практика управления схемой базы данных в production*
