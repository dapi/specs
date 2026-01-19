# SRG-009 Database Migrations (Database Schema Management in Production)

Database Migrations is the practice of managing database schema and data through ordered changes, allowing the database structure to evolve with the application while maintaining data integrity in production environments.

---

## Migration Anti-Patterns ❌

### ❌ Manual Schema Changes
```bash
# DO NOT DO THIS!
$ psql production_db
> ALTER TABLE users ADD COLUMN email VARCHAR(255);  # Without backup
> DELETE FROM logs WHERE created_at < '2024-01-01';  # Without WHERE check
```
**Risks:**
- Errors in production
- Impossible to rollback
- Data loss
- Inconsistency between environments

### ❌ Without Migrations (Manual Changes)
```sql
-- Database in each environment - manual work
-- Impossible to track versions
-- Impossible to rollback
```

### ❌ Without Backups
```bash
# Only copy of database is production
# No restore procedure testing
```

### ❌ Migration Execution Without Downtime Planning
```bash
# Running migration during peak hours
# No plan B
```

---

## Migration Management Principles ✅

### 1. Schema Versioning

```bash
# Repository structure
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
└── schema.rb  # or schema.sql
```

### 2. Idempotent Migrations

```sql
-- 003_add_user_email.sql
-- Idempotent migration - can be run multiple times

-- Check if column doesn't exist
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

### 3. Backups (Always backup first!)

```bash
#!/bin/bash
# backup-and-migrate.sh

# Create backup
pg_dump -h $DB_HOST -U $DB_USER -d $DB_NAME -F c -f backup-$(date +%Y%m%d_%H%M%S).bak

# Validate backup
if ! pg_restore --list backup-*.bak | head -10; then
    echo "❌ Backup validation failed!"
    exit 1
fi

echo "✅ Backup created and validated"
```

---

## Migration Types

### Type 1: Expand/Contract Pattern (Zero-downtime Migrations)

```sql
-- Step 1: EXPAND - Add new column
-- Migration: 005_add_new_column.sql
-- Downtime: None
-- Backward compatible: Yes

START TRANSACTION;

ALTER TABLE users
ADD COLUMN email_verified BOOLEAN DEFAULT false;

-- Create trigger for data synchronization
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
-- Step 2: DUAL WRITE - Write to both columns
-- Application updated to write to both columns
-- Old application versions still work

-- Application v1: Writes to verified_at
-- Application v2: Writes to both verified_at and email_verified

-- Data migration (in background)
UPDATE users
SET email_verified = (verified_at IS NOT NULL)
WHERE email_verified IS NULL;
```

```sql
-- Step 3: CONTRACT - Remove old column
-- Migration: 006_remove_old_column.sql
-- Downtime: None (after all instances are updated)
-- Backward compatible: No (after deployment)

-- First disable trigger
DROP TRIGGER users_sync_verified ON users;
DROP FUNCTION sync_email_verified();

-- Remove old column
-- ALTER TABLE users DROP COLUMN verified_at;
```

### Type 2: Add-only Migrations (Safe)

```sql
-- Only add, never delete
-- Can always rollback

-- Add column (can be NULL or has DEFAULT)
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Create new table
CREATE TABLE user_profiles (...);

-- Add new index
CREATE INDEX CONCURRENTLY idx_users_phone ON users(phone);
-- CONCURRENTLY = without table lock
```

### Type 3: Heavy Migrations (Split and Conquer)

```python
#!/usr/bin/env python3
# large_migration.py

# For large tables (millions of records)
# Migrate in batches to minimize locks

import psycopg2
import time

BATCH_SIZE = 10000
SLEEP_BETWEEN_BATCHES = 0.5  # seconds

def migrate_large_table():
    conn = psycopg2.connect(os.getenv('DATABASE_URL'))
    cursor = conn.cursor()

    # Migrate in batches
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
-- Populate new column with data from existing columns

-- Approach 1: Online backfill (in background)
-- Run after adding column

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
            PERFORM pg_sleep(0.1);  -- Small pause
        ELSE
            EXIT;
        END IF;
    END LOOP;

    RAISE NOTICE 'Updated % rows', total_updated;
END;
$$ LANGUAGE plpgsql;

-- Run in background process
SELECT backfill_users_name();
```

---

## Migration Frameworks by Language

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

# Execution
$ rails db:migrate
$ rails db:rollback  # Rollback last migration
$ rails db:migrate:down VERSION=20240115000001  # Rollback specific
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

# Execution
$ python manage.py migrate
$ python manage.py migrate users 0011  # Rollback
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

// Execution
$ npx sequelize-cli db:migrate
$ npx sequelize-cli db:migrate:undo  # Rollback last
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
# Create migration
$ migrate create -ext sql -dir db/migrations add_email_to_users

# File contents:
# db/migrations/001_add_email_to_users.up.sql
ALTER TABLE users ADD COLUMN email VARCHAR(255);
CREATE INDEX idx_users_email ON users(email);

# db/migrations/001_add_email_to_users.down.sql
DROP INDEX idx_users_email;
ALTER TABLE users DROP COLUMN email;

# Execution
$ migrate -database $DATABASE_URL -path db/migrations up
$ migrate -database $DATABASE_URL -path db/migrations down 1
```

---

## Migration Deployment Strategies

### Strategy 1: Deploy → Migrate

```bash
# Approach: Deploy code first, then migration
# Risks: Application may not start
# When: For backward compatible migrations

# Step 1: Deploy new code
$ kubectl rollout restart deployment/app --image=new-version

# Step 2: Run migrations
$ kubectl run migration --rm -i --tty --image=app-migration \
  -- ./migrate up
```

### Strategy 2: Migrate → Deploy

```bash
# Approach: Migration first, then code
# Safer, but requires planning

# Step 1: Run migrations
$ ./migrate up

# Step 2: Verify migrations
$ ./migrate verify

# Step 3: Deploy code (after successful migration)
$ kubectl rollout restart deployment/app --image=new-version
```

### Strategy 3: Rolling Migrations (for zero-downtime)

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
        # Migrations run before pod starts
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

## Unsafe Operations and Safe Alternatives

### ❌ Unsafe: ALTER TABLE on Large Tables
```sql
ALTER TABLE million_rows_table ADD COLUMN new_col VARCHAR(255);
-- Blocks table for minutes/hours
-- Records cannot be inserted/updated/deleted
```

### ✅ Safe: Using pg_repack or logical replication
```bash
# Install pg_repack
$ pg_repack -d mydb -t users \
  --alter 'ADD COLUMN new_col VARCHAR(255)'

# Concurrently recreates table
# Without locks
# Can be cancelled
```

### ❌ Unsafe: DROP COLUMN
```sql
ALTER TABLE users DROP COLUMN old_column;
-- Data loss forever
-- Difficult to rollback
-- Requires heavy I/O
```

### ✅ Safe: Mark as deprecated first
```sql
-- Step 1: Mark column as deprecated
COMMENT ON COLUMN users.old_column IS 'DEPRECATED - Remove after 2024/Q2';

-- Step 2: Set column to NULL to free space
UPDATE users SET old_column = NULL WHERE old_column IS NOT NULL;

-- Step 3: Remove after verifying no code uses it
-- ALTER TABLE users DROP COLUMN old_column;
```

### ❌ Unsafe: UPDATE without WHERE
```sql
UPDATE users SET status = 'active';
-- Updates all records
-- Better to add WHERE clause
```

### ✅ Safe: Batch updates
```sql
-- Update in batches
UPDATE users
SET status = 'active'
WHERE id BETWEEN 1 AND 10000
  AND status IS NULL;

-- Check count
SELECT COUNT(*) FROM users WHERE status IS NULL;

-- Repeat until 0
```

---

## Migration Monitoring

```python
class MigrationMonitoring:
    def monitor_migration(self, migration_id):
        """Monitor running migration"""

        # Check duration and locks
        locks = self.db.query("""
            SELECT * FROM pg_locks
            WHERE NOT granted
              AND relation IN (
                SELECT oid FROM pg_class WHERE relname = 'users'
              )
        """)

        if locks:
            self.send_alert('Migration causing locks', severity='warning')

        # Check deadlocks
        deadlocks = self.db.query("""
            SELECT * FROM pg_stat_database
            WHERE deadlocks > 0
        """)

        if deadlocks:
            self.send_alert('Deadlocks detected during migration', severity='critical')

        # Cancel migration if too long
        duration = self.get_migration_duration(migration_id)
        if duration > timedelta(minutes=30):
            self.cancel_migration(migration_id)
            self.send_alert('Migration cancelled due to timeout')
```

---

## Backup Before Migrations

### Automated Backup

```bash
#!/bin/bash
# backup-db.sh

set -e

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DB_NAME="${DB_NAME:-myapp_production}"
BACKUP_BUCKET="${BACKUP_BUCKET:-myapp-backups}"

echo "🔒 Creating backup of $DB_NAME..."

# Create backup
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

# Validate backup
echo "🔍 Validating backup..."
pg_restore --list /tmp/db_backup_$TIMESTAMP.dump | head -20
echo "✅ Backup validated"

# Compress
gzip /tmp/db_backup_$TIMESTAMP.dump
echo "✅ Backup compressed"

# Upload to S3
aws s3 cp /tmp/db_backup_$TIMESTAMP.dump.gz s3://$BACKUP_BUCKET/backups/
echo "✅ Backup uploaded to S3"

# Log in database
echo "📝 Logging backup..."
psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c \
  "INSERT INTO db_backups (file_name, created_at, size_bytes) VALUES
   ('db_backup_$TIMESTAMP.dump.gz', NOW(), $(stat -c%s /tmp/db_backup_$TIMESTAMP.dump.gz));"

# Clean old backups (keep last 7 days)
find /tmp -name "db_backup_*.dump.gz" -mtime +7 -delete

# Move to archive in S3
aws s3 mv s3://$BACKUP_BUCKET/backups/db_backup_$TIMESTAMP.dump.gz \
       s3://$BACKUP_BUCKET/archive/

echo "🎉 Backup process complete!"
```

---

## Testing Rollback

```bash
#!/bin/bash
# test-rollback.sh

# 1. Create test database
psql -c "CREATE DATABASE test_restore;"

# 2. Restore backup
pg_restore -d test_restore /tmp/db_backup_test.dump

# 3. Run application tests
./run-tests.sh --database=test_restore

# 4. Rollback migration
./migrate down

# 5. Check that application works
./run-tests.sh --database=test_restore

# 6. Cleanup
dropdb test_restore

echo "✅ Rollback test passed!"
```

---

## Configuration via Environment Variables

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

## Best Practices ✅

### Migration Storage
- ✅ Version in Git
- ✅ Explicit naming (YYYYMMMDD_description.sql)
- ✅ Store up/down pairs
- ✅ Minimize manual edits

### Execution
- ✅ Test on dev/staging
- ✅ Run during off-peak hours
- ✅ Execute upgrades with downgrade plan
- ✅ Track duration
- ✅ Execute in transactions (when possible)

### Observability
- ✅ Monitor locks
- ✅ Monitor I/O
- ✅ Watch disk space
- ✅ Log errors
- ✅ Alert on long migrations

### Safety
- ✅ Backup before every migration
- ✅ Have rollback plan
- ✅ Don't migrate directly to production
- ✅ Have emergency procedures

---

*Database Migrations - database schema management practice*
