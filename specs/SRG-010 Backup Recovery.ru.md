# SRG-010 Backup & Recovery (Резервное копирование и восстановление данных)

Backup & Recovery - это комплексная стратегия создания резервных копий данных, их хранения, проверки и восстановления для защиты от потери данных, обеспечения непрерывности бизнеса и выполнения требований к хранению информации.

---

## Виды резервных копий

### 1. Full Backup (Полное резервное копирование)

**Описание:** Копия всех данных в системе.

**Плюсы:**
- Простое восстановление (один файл)
- Полная целостность данных
- Быстрый restore для небольших объемов

**Минусы:**
- Долго создается
- Занимает много места
- Высокая нагрузка на систему

**Когда использовать:**
- Первоначальное резервное копирование
- Регулярно (раз в неделю/месяц)
- Для критичных данных

**Пример:**
```bash
#!/bin/bash
# full_backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/full"
S3_BUCKET="s3://myapp-backups"

echo "🔒 Starting full backup: $DATE"

# PostgreSQL
pg_dump -h $DB_HOST -U $DB_USER -d $DB_NAME \
  --verbose \
  --no-password \
  --clean \
  --if-exists \
  --no-owner \
  --no-privileges \
  --no-acl \
  --file $BACKUP_DIR/full_$DATE.sql

# Проверка целостности
if pg_restore --list $BACKUP_DIR/full_$DATE.sql > /dev/null 2>&1; then
  echo "✅ Backup validation passed"

  # Сжатие
  gzip $BACKUP_DIR/full_$DATE.sql

  # Загрузка в S3
  aws s3 cp $BACKUP_DIR/full_$DATE.sql.gz $S3_BUCKET/full/

  # Проверка в S3
  aws s3 ls $S3_BUCKET/full/full_$DATE.sql.gz

  echo "🎉 Full backup completed successfully"
else
  echo "❌ Backup validation failed"
  exit 1
fi
```

---

### 2. Incremental Backup (Инкрементное резервное копирование)

**Описание:** Копирует только измененные данные с момента последней резервной копии (full или incremental).

**Плюсы:**
- Быстрое создание
- Меньше занимает места
- Меньше нагрузки на систему

**Минусы:**
- Сложное восстановление (необходимо применить цепочку инкрементных копий)
- Если одна копия повреждена — все последующие бесполезны

**Пример:**
```bash
#!/bin/bash
# incremental_backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/incremental"
S3_BUCKET="s3://myapp-backups"

# Дата последней полной копии
LAST_FULL=$(ls -t /backups/full/full_*.sql.gz | head -1 | sed 's/.*full_//' | sed 's/.sql.gz//')

echo "🔒 Starting incremental backup from $LAST_FULL"

# PostgreSQL с WAL (Write-Ahead Logging)
# Создаем базовую полную копию
gpg --basebackup -h $DB_HOST -U $DB_USER -D $BACKUP_DIR/base/

# Создаем архив WAL
psql -h $DB_HOST -U $DB_USER -c "SELECT pg_switch_wal();"

# Копируем WAL файлы
find /var/lib/postgresql/wal/ -newer $BACKUP_DIR/base/ -exec cp {} $BACKUP_DIR/incremental/ \;

# Сжатие
tar -czf $BACKUP_DIR/incremental_$DATE.tar.gz $BACKUP_DIR/incremental/

# Загрузка в S3
aws s3 cp $BACKUP_DIR/incremental_$DATE.tar.gz $S3_BUCKET/incremental/

echo "🎉 Incremental backup completed"
```

---

### 3. Differential Backup (Дифференциальное резервное копирование)

**Описание:** Копирует все измененные данные с момента последней полной копии.

**Плюсы:**
- Быстрое создание (меньше чем full, больше чем incremental)
- Быстрое восстановление (нужен только full + последний differential)
- Более надежное чем incremental

**Минусы:**
- Занимает больше места чем incremental
- Размер растет по мере накопления изменений

**Пример:**
```bash
#!/bin/bash
# differential_backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/differential"
S3_BUCKET="s3://myapp-backups"

# Дата последней полной копии
LAST_FULL_DATE=$(ls -t /backups/full/full_*.sql.gz | head -1 | sed 's/.*full_//' | sed 's/.sql.gz//')

echo "🔒 Starting differential backup from last full ($LAST_FULL_DATE)"

# MySQL example
mysqldump -h $DB_HOST -u $DB_USER -p$DB_PASS \
  --single-transaction \
  --master-data=2 \
  --flush-logs \
  --ignore-table=mysql.innodb_index_stats \
  --ignore-table=mysql.innodb_table_stats \
  --where="updated_at >= '$LAST_FULL_DATE'" \
  $DB_NAME > $BACKUP_DIR/differential_$DATE.sql

# Сжатие
gzip $BACKUP_DIR/differential_$DATE.sql

# Загрузка в S3
aws s3 cp $BACKUP_DIR/differential_$DATE.sql.gz $S3_BUCKET/differential/

echo "🎉 Differential backup completed"
```

---

### 4. Continuous Backup / Point-in-Time Recovery (PITR)

**Описание:** Непрерывное резервное копирование журналов транзакций (WAL в PostgreSQL, binlog в MySQL).

**Плюсы:**
- Восстановление до любой точки во времени
- Минимальная потеря данных
- Автоматический процесс

**Минусы:**
- Сложная настройка
- Требует хранения больших журналов
- Зависит от базовой полной копии

**PostgreSQL PITR:**
```bash
# 1. Настройка postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /var/lib/postgresql/wal/%f && cp %p /var/lib/postgresql/wal/%f'

# 2. Создание base backup
pg_basebackup -h $DB_HOST -U $DB_USER -D /backups/base/ -P -v

# 3. Архивирование WAL
cd /var/lib/postgresql/wal/
find . -type f -name "*.wal" -exec gzip {} \;
aws s3 sync . s3://myapp-backups/wal/

# 4. Восстановление до точки во времени
# recovery.conf
restore_command = 'cp s3://myapp-backups/wal/%f %p'
recovery_target_time = '2024-01-15 14:30:00'
recovery_target_action = 'promote'
```

---

## Расписание резервного копирования

### Пример расписания

```yaml
backup_schedule:
  full_backup:
    frequency: "weekly"
    day: "sunday"
    time: "02:00 AM"  # Off-peak hours
    retention: "90 days"

  differential_backup:
    frequency: "daily"
    time: "02:00 AM"
    retention: "30 days"

  incremental_backup:
    frequency: "hourly"
    retention: "7 days"

  continuous_backup:
    frequency: "real-time"
    wal_archive: "continuous"
    retention: "14 days"

  automated_backup_verification:
    frequency: "daily"
    time: "06:00 AM"
    action: "restore to test environment"
```

---

## Хранение резервных копий

### 3-2-1 Rule (золотой стандарт)

```yaml
storage_strategy:
  rule_3_2_1:
    3_copies: true  # Три копии данных
    2_different_media: true  # Два разных типа носителя
      # 1: Оригинал (production)
      # 2: Резервная копия (диски/HDD)
      # 3: Резервная копия (cloud/offline)
    1_offsite: true  # Одна копия внешнего хранения

  locations:
    primary:
      type: "local_disks"
      path: "/backups"
      retention: "7 days"
      access_speed: "fast"

    secondary:
      type: "network_attached_storage"
      path: "nas://10.0.1.100/backups"
      retention: "30 days"
      access_speed: "medium"

    tertiary:
      type: "cloud_storage"
      provider: "aws_s3"
      bucket: "myapp-backups"
      region: "us-west-2"
      storage_class: "glacier_deep_archive"  # Для данных старше 90 дней
      retention: "7 years"  # Compliance requirements
      access_speed: "slow"
      cost: "$0.00099/GB/month"
```

---

## Восстановление

### Стратегии восстановления

#### Strategy 1: Full Restore (Full + Differential + Incremental)

```bash
#!/bin/bash
# restore_full.sh

RESTORE_DATE="2024-01-15"
RESTORE_DIR="/restore/$RESTORE_DATE"
S3_BUCKET="s3://myapp-backups"

mkdir -p $RESTORE_DIR

echo "🔄 Restoring database to $RESTORE_DATE"

# 1. Скачиваем полную копию
aws s3 cp $S3_BUCKET/full/full_20240114_020000.sql.gz $RESTORE_DIR/
gunzip $RESTORE_DIR/full_20240114_020000.sql.gz

# 2. Восстанавливаем полную копию
psql -h $DB_HOST -U $DB_USER -d postgres -c "DROP DATABASE IF EXISTS myapp_restore;"
psql -h $DB_HOST -U $DB_USER -d postgres -c "CREATE DATABASE myapp_restore;"

psql -h $DB_HOST -U $DB_USER -d myapp_restore \
  < $RESTORE_DIR/full_20240114_020000.sql

# 3. Скачиваем и применяем дифференциальные копии
for diff in $(aws s3 ls $S3_BUCKET/differential/ --recursive | grep "20240114" | awk '{print $4}'); do
  aws s3 cp $S3_BUCKET/$diff $RESTORE_DIR/
  gunzip $RESTORE_DIR/$(basename $diff)
  psql -h $DB_HOST -U $DB_USER -d myapp_restore \
    < $RESTORE_DIR/$(basename $diff .gz)
done

# 4. Скачиваем и применяем инкрементальные копии
for inc in $(aws s3 ls $S3_BUCKET/incremental/ --recursive | grep "20240115" | awk '{print $4}'); do
  aws s3 cp $S3_BUCKET/$inc $RESTORE_DIR/
  tar -xzf $RESTORE_DIR/$(basename $inc)
  psql -h $DB_HOST -U $DB_USER -d myapp_restore \
    -f $RESTORE_DIR/incremental/wal_*.sql
done

echo "✅ Restore completed to myapp_restore"
```

#### Strategy 2: Point-in-Time Recovery (PITR)

```bash
#!/bin/bash
# restore_pitr.sh

TARGET_TIME="2024-01-15 14:30:00"
RESTORE_DIR="/restore/pitr"
S3_BUCKET="s3://myapp-backups"

echo "🔄 Point-in-Time Recovery to $TARGET_TIME"

# 1. Останавливаем PostgreSQL
systemctl stop postgresql

# 2. Скачиваем base backup
aws s3 cp $S3_BUCKET/base/base_20240114.tar.gz $RESTORE_DIR/
tar -xzf $RESTORE_DIR/base_20240114.tar.gz -C $RESTORE_DIR/

# 3. Настройка recovery
cat > $RESTORE_DIR/recovery.conf <<EOF
restore_command = 'aws s3 cp $S3_BUCKET/wal/%f %p'
recovery_target_time = '$TARGET_TIME'
recovery_target_action = 'promote'
recovery_target_inclusive = false
EOF

# 4. Запускаем PostgreSQL с recovery
systemctl start postgresql

# 5. Следим за логами
while true; do
  if grep -q "database system is ready to accept connections" /var/log/postgresql/postgresql-14-main.log; then
    echo "✅ Database recovered to $TARGET_TIME"
    break
  fi
  sleep 10
done
```

#### Strategy 3: Table-level Restore

```bash
#!/bin/bash
# restore_table.sh

TABLE_NAME="users"
RESTORE_DATE="2024-01-15"
RESTORE_DIR="/restore/table"
S3_BUCKET="s3://myapp-backups"

echo "🔄 Restoring table $TABLE_NAME to $RESTORE_DATE"

# 1. Создаем временную базу для восстановления
psql -h $DB_HOST -U $DB_USER -d postgres -c "DROP DATABASE IF EXISTS temp_restore;"
psql -h $DB_HOST -U $DB_USER -d postgres -c "CREATE DATABASE temp_restore;"

# 2. Восстанавливаем полную копию во временную базу
aws s3 cp $S3_BUCKET/full/full_20240114_020000.sql.gz $RESTORE_DIR/
gunzip $RESTORE_DIR/full_20240114_020000.sql.gz

psql -h $DB_HOST -U $DB_USER -d temp_restore \
  < $RESTORE_DIR/full_20240114_020000.sql

# 3. Экспортируем нужную таблицу
pg_dump -h $DB_HOST -U $DB_USER -d temp_restore \
  --table=$TABLE_NAME \
  --data-only \
  > $RESTORE_DIR/table_$TABLE_NAME.sql

# 4. Восстанавливаем таблицу в production
psql -h $DB_HOST -U $DB_USER -d myapp_production \
  < $RESTORE_DIR/table_$TABLE_NAME.sql

# 5. Очищаем временную базу
psql -h $DB_HOST -U $DB_USER -d postgres -c "DROP DATABASE temp_restore;"

echo "✅ Table $TABLE_NAME restored successfully"
```

---

## Проверка резервных копий (Backup Verification)

### Автоматическая проверка

```python
#!/usr/bin/env python3
# verify_backup.py

import subprocess
import psycopg2
import os
from datetime import datetime, timedelta

def verify_latest_backup():
    """Проверяем последнюю резервную копию"""

    # Находим последнюю копию
    latest_backup = get_latest_backup()
    print(f"🔍 Verifying backup: {latest_backup}")

    # 1. Проверяем целостность файла
    if not verify_file_integrity(latest_backup):
        send_alert("Backup file corrupted", severity="critical")
        return False

    print("✅ File integrity check passed")

    # 2. Восстанавливаем в тестовую базу
    test_db_name = "verify_backup_" + datetime.now().strftime("%Y%m%d_%H%M%S")
    create_test_database(test_db_name)

    try:
        if not restore_to_test(latest_backup, test_db_name):
            send_alert("Restore verification failed", severity="critical")
            return False

        print("✅ Restore test passed")

        # 3. Запускаем базовые тесты
        if not run_basic_tests(test_db_name):
            send_alert("Backup data validation failed", severity="critical")
            return False

        print("✅ Data validation tests passed")

        # 4. Проверяем актуальность
        backup_age = get_backup_age(latest_backup)
        if backup_age > timedelta(hours=25):
            send_alert(f"Backup is too old: {backup_age}", severity="warning")

        print(f"✅ Backup age: {backup_age} (good)")

        send_alert("Backup verification successful", severity="info")
        return True

    finally:
        cleanup_test_database(test_db_name)

def get_latest_backup():
    """Находим последнюю резервную копию"""
    result = subprocess.run(
        ["aws", "s3", "ls", "s3://myapp-backups/full/", "--recursive"],
        capture_output=True,
        text=True
    )

    files = result.stdout.strip().split('\n')
    latest = sorted(files, reverse=True)[0]
    return latest.split()[-1]

def verify_file_integrity(backup_path):
    """Проверяем целостность файла"""
    # Скачиваем файл
    local_path = f"/tmp/{os.path.basename(backup_path)}"
    subprocess.run(["aws", "s3", "cp", f"s3://myapp-backups/{backup_path}", local_path])

    # Проверяем контрольную сумму
    result = subprocess.run(["md5sum", local_path], capture_output=True, text=True)
    md5 = result.stdout.split()[0]

    # Сравниваем с ранее сохраненной контрольной суммой
    with open(f"{local_path}.md5", "r") as f:
        expected_md5 = f.read().strip()

    os.remove(local_path)
    return md5 == expected_md5

def restore_to_test(backup_path, test_db_name):
    """Восстанавливаем в тестовую базу"""
    local_path = f"/tmp/{os.path.basename(backup_path)}"

    # Скачиваем и распаковываем
    subprocess.run(["aws", "s3", "cp", f"s3://myapp-backups/{backup_path}", local_path])
    subprocess.run(["gunzip", local_path])

    # Восстанавливаем
    result = subprocess.run(
        ["pg_restore", "-h", os.getenv("DB_HOST"), "-U", os.getenv("DB_USER"), "-d", test_db_name, local_path[:-3]],
        capture_output=True
    )

    os.remove(local_path[:-3])  # .dump file
    return result.returncode == 0

def run_basic_tests(db_name):
    """Запускаем базовые тесты на восстановленных данных"""
    conn = psycopg2.connect(
        host=os.getenv("DB_HOST"),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
        database=db_name
    )

    cursor = conn.cursor()

    try:
        # Тест 1: Проверяем подключение
        cursor.execute("SELECT 1")
        assert cursor.fetchone()[0] == 1

        # Тест 2: Проверяем количество пользователей
        cursor.execute("SELECT COUNT(*) FROM users")
        user_count = cursor.fetchone()[0]
        assert user_count > 0, "No users found"

        # Тест 3: Проверям индексы
        cursor.execute("""
            SELECT tablename, indexname
            FROM pg_indexes
            WHERE schemaname = 'public'
        """)
        indexes = cursor.fetchall()
        assert len(indexes) > 0, "No indexes found"

        # Тест 4: Проверяем foreign keys
        cursor.execute("""
            SELECT conname
            FROM pg_constraint
            WHERE contype = 'f'
        """)

        return True

    except Exception as e:
        print(f"Test failed: {e}")
        return False

    finally:
        cursor.close()
        conn.close()

def get_backup_age(backup_path):
    """Получаем время создания резервной копии"""
    result = subprocess.run(
        ["aws", "s3", "ls", f"s3://myapp-backups/{backup_path}"],
        capture_output=True,
        text=True
    )

    date_str = result.stdout.split()[0]  # YYYY-MM-DD
    time_str = result.stdout.split()[1]  # HH:MM:SS

    backup_time = datetime.strptime(f"{date_str} {time_str}", "%Y-%m-%d %H:%M:%S")
    return datetime.now() - backup_time

def create_test_database(db_name):
    """Создаем тестовую базу"""
    subprocess.run([
        "psql", "-h", os.getenv("DB_HOST"), "-U", os.getenv("DB_USER"), "-d", "postgres",
        "-c", f"CREATE DATABASE {db_name};"
    ], check=True)

def cleanup_test_database(db_name):
    """Очищаем тестовую базу"""
    subprocess.run([
        "psql", "-h", os.getenv("DB_HOST"), "-U", os.getenv("DB_USER"), "-d", "postgres",
        "-c", f"DROP DATABASE IF EXISTS {db_name};"
    ], check=True)

def send_alert(message, severity):
    """Отправляем alert"""
    webhook_url = os.getenv("SLACK_WEBHOOK")

    payload = {
        "text": f"[Backup Verification] {severity.upper()}: {message}",
        "channel": "#alerts",
        "username": "Backup Bot"
    }

    import requests
    requests.post(webhook_url, json=payload)

if __name__ == "__main__":
    verify_latest_backup()
```

---

## Восстановление после инцидента

### Disaster Recovery Plan

```yaml
disaster_recovery:
  rto: "4 hours"  # Recovery Time Objective
  rpo: "15 minutes"  # Recovery Point Objective

  scenarios:
    hardware_failure:
      priority: high
      steps:
        1: "Activate standby database"
        2: "Redirect traffic to standby"
        3: "Restore from latest backup"
        4: "Apply WAL/binlogs for PITR"
        5: "Verify data integrity"
        6: "Promote standby to primary"

    data_corruption:
      priority: critical
      steps:
        1: "Stop all writes immediately"
        2: "Identify corruption time"
        3: "Restore to time before corruption"
        4: "Validate restored data"
        5: "Gradual traffic restoration"

    ransomware:
      priority: critical
      steps:
        1: "Isolate affected systems"
        2: "Assess damage scope"
        3: "Restore from offline backups"
        4: "Security audit and patching"
        5: "Gradual restoration"

    region_failure:
      priority: high
      steps:
        1: "Activate multi-region failover"
        2: "Restore cross-region backups"
        3: "Update DNS and routing"
        4: "Verify all services"
```

---

## Стоимость хранения

```yaml
storage_cost_analysis:
  monthly_costs:
    local_ssd:
      size_tb: 2
      cost_per_tb: $50
      monthly: $100

    nas:
      size_tb: 10
      cost_per_tb: $20
      monthly: $200

    aws_s3_standard:
      size_tb: 50
      cost_per_tb: $23
      monthly: $1150

    aws_s3_glacier_deep:
      size_tb: 500
      cost_per_tb: $0.99
      monthly: $495

  total_monthly: $1945
  total_annually: $23340

  optimization_strategies:
    - tiered_storage: "Move old backups to cheaper storage"
    - compression: "Use zstd/bzip2 instead of gzip"
    - deduplication: "Avoid duplicate backups"
    - retention_policies: "Delete old backups automatically"
```

---

## Автоматизация

### Backup Orchestration

```yaml
# backup-orchestration.yml
tasks:
  - name: "Full Backup - Sunday 2 AM"
    schedule: "0 2 * * 0"  # cron: every Sunday at 2 AM
    type: full
    actions:
      - notify_team: "Starting weekly full backup"
      - create_backup: {}
      - verify_backup: {}
      - upload_to_s3: {storage_class: standard}
      - update_catalog: {}
      - cleanup_local: {retention: 7 days}
      - notify_team: "Backup completed successfully"
    on_failure:
      - alert_pager: severity=high
      - retry: {attempts: 3, delay: 1h}

  - name: "Incremental Backup - Daily 2 AM"
    schedule: "0 2 * * 1-6"  # Monday-Saturday
    type: incremental
    actions:
      - create_backup: {}
      - verify_backup: {}
      - upload_to_s3: {storage_class: standard_ia}
      - cleanup_local: {retention: 3 days}

  - name: "Verify Latest Backup - Daily 6 AM"
    schedule: "0 6 * * *"  # Every day at 6 AM
    type: verification
    actions:
      - download_latest: {}
      - verify_integrity: {}
      - restore_to_test: {}
      - run_tests: {}
      - cleanup_test: {}

  - name: "Monthly Archival - 1st of month"
    schedule: "0 3 1 * *"  # 1st day of month at 3 AM
    type: archival
    actions:
      - find_old_backups: {older_than: 90 days}
      - move_to_glacier: {}
      - update_catalog: {}
```

---

## Конфигурация

```bash
# .env.backups
BACKUP_STRATEGY=full_differential_incremental
FULL_BACKUP_DAY=sunday
FULL_BACKUP_TIME=02:00
DIFFERENTIAL_TIME=02:00
INCREMENTAL_INTERVAL=hourly

# Database
DB_TYPE=postgresql
DB_HOST=prod-db.example.com
DB_USER=backup_user
DB_PASSWORD=$ecret
DB_NAME=myapp_production

# Storage
LOCAL_BACKUP_PATH=/backups
S3_BUCKET=myapp-backups
S3_REGION=us-west-2
S3_STORAGE_CLASS=standard

# Retention
RETENTION_FULL_DAYS=30
RETENTION_DIFF_DAYS=14
RETENTION_INC_DAYS=7
RETENTION_WAL_DAYS=14

# Notifications
SLACK_WEBHOOK=https://hooks.slack.com/services/xxx
PAGERDUTY_INTEGRATION_KEY=xxxxx
EMAIL_ALERTS=ops-team@example.com

# Security
ENCRYPTION_KEY=base64_encoded_key
SIGNATURE_ALGORITHM=SHA256
```

---

## Best Practices ✅

### Хранение
- ✅ 3-2-1 rule (3 копии, 2 носителя, 1 внешний)
- ✅ Проверяйте бэкапы автоматически каждый день
- ✅ Тестируйте process восстановления каждый месяц
- ✅ Используйте шифрование (at rest и in transit)
- ✅ Регулярно ротируйте ключи шифрования
- ✅ Сохраняйте проверочные суммы (md5/sha256)

### Восстановление
- ✅ Документируйте все процедуры восстановления
- ✅ Практикуйте disaster recovery каждый квартал
- ✅ Измеряйте RTO и RPO
- ✅ Поддерживайте runbooks в актуальном состоянии
- ✅ Имейте clear ownership

### Безопасность
- ✅ RBAC для доступа к бэкапам
- ✅ Audit logs всех операций
- ✅ MFA для доступа к хранилищу
- ✅ Network isolation для бэкап-серверов
- ✅ Regular penetration testing

### Процессы
- ✅ Review and test disaster recovery plan quarterly
- ✅ Update runbooks after каждого инцидента
- ✅ Train new team members на процедуры восстановления
- ✅ Conduct blameless postmortems для failures
- ✅ Automate где возможно

---

## Инструменты

### Cloud-Native Tools
- **AWS Backup**: Managed backup service
- **AWS RDS Snapshots**: Automated DB backups
- **GCP Cloud Backup**: For GCP resources
- **Azure Backup**: For Azure services

### Database-Specific
- **pg_dump / pg_basebackup**: PostgreSQL
- **mysqldump / XtraBackup**: MySQL/MariaDB
- **mongodump**: MongoDB
- **redis-cli --rdb**: Redis
- **cassandra snapshots**: Cassandra

### Enterprise Solutions
- **Veeam**: Virtual, physical, cloud
- **Commvault**: Data protection platform
- **Rubrik**: Cloud data management
- **Cohesity**: Hyperconverged backup

### Open-Source
- **Borg**: Deduplicating archiver
- **Restic**: Fast, secure backups
- **Duplicati**: Free backup software
- **Amanda**: Advanced Maryland Automatic Network Disk Archiver

---

*Backup & Recovery - практика создания резервных копий данных и восстановления для обеспечения непрерывности бизнеса*
