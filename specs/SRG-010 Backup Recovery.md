# SRG-010 Backup & Recovery

Backup & Recovery is a comprehensive strategy for creating data backups, storing them, verifying, and restoring to protect against data loss, ensure business continuity, and meet information retention requirements.

---

## Backup Types

### 1. Full Backup

**Description:** Copy of all data in the system.

**Pros:**
- Simple restore (single file)
- Complete data integrity
- Fast restore for small volumes

**Cons:**
- Takes long to create
- Takes up a lot of space
- High system load

**When to use:**
- Initial backup
- Regularly (weekly/monthly)
- For critical data

**Example:**
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

# Integrity check
if pg_restore --list $BACKUP_DIR/full_$DATE.sql > /dev/null 2>&1; then
  echo "✅ Backup validation passed"

  # Compression
  gzip $BACKUP_DIR/full_$DATE.sql

  # Upload to S3
  aws s3 cp $BACKUP_DIR/full_$DATE.sql.gz $S3_BUCKET/full/

  # Verify in S3
  aws s3 ls $S3_BUCKET/full/full_$DATE.sql.gz

  echo "🎉 Full backup completed successfully"
else
  echo "❌ Backup validation failed"
  exit 1
fi
```

---

### 2. Incremental Backup

**Description:** Copies only changed data since the last backup (full or incremental).

**Pros:**
- Fast creation
- Takes less space
- Less system load

**Cons:**
- Complex restore (need to apply chain of incremental copies)
- If one copy is corrupted — all subsequent are useless

**Example:**
```bash
#!/bin/bash
# incremental_backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/incremental"
S3_BUCKET="s3://myapp-backups"

# Date of last full backup
LAST_FULL=$(ls -t /backups/full/full_*.sql.gz | head -1 | sed 's/.*full_//' | sed 's/.sql.gz//')

echo "🔒 Starting incremental backup from $LAST_FULL"

# PostgreSQL with WAL (Write-Ahead Logging)
# Create base full copy
gpg --basebackup -h $DB_HOST -U $DB_USER -D $BACKUP_DIR/base/

# Create WAL archive
psql -h $DB_HOST -U $DB_USER -c "SELECT pg_switch_wal();"

# Copy WAL files
find /var/lib/postgresql/wal/ -newer $BACKUP_DIR/base/ -exec cp {} $BACKUP_DIR/incremental/ \;

# Compression
tar -czf $BACKUP_DIR/incremental_$DATE.tar.gz $BACKUP_DIR/incremental/

# Upload to S3
aws s3 cp $BACKUP_DIR/incremental_$DATE.tar.gz $S3_BUCKET/incremental/

echo "🎉 Incremental backup completed"
```

---

### 3. Differential Backup

**Description:** Copies all changed data since the last full backup.

**Pros:**
- Fast creation (less than full, more than incremental)
- Fast restore (need only full + latest differential)
- More reliable than incremental

**Cons:**
- Takes more space than incremental
- Size grows as changes accumulate

**Example:**
```bash
#!/bin/bash
# differential_backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/differential"
S3_BUCKET="s3://myapp-backups"

# Date of last full backup
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

# Compression
gzip $BACKUP_DIR/differential_$DATE.sql

# Upload to S3
aws s3 cp $BACKUP_DIR/differential_$DATE.sql.gz $S3_BUCKET/differential/

echo "🎉 Differential backup completed"
```

---

### 4. Continuous Backup / Point-in-Time Recovery (PITR)

**Description:** Continuous backup of transaction logs (WAL in PostgreSQL, binlog in MySQL).

**Pros:**
- Restore to any point in time
- Minimal data loss
- Automatic process

**Cons:**
- Complex setup
- Requires storage of large logs
- Depends on base full backup

**PostgreSQL PITR:**
```bash
# 1. Configure postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /var/lib/postgresql/wal/%f && cp %p /var/lib/postgresql/wal/%f'

# 2. Create base backup
pg_basebackup -h $DB_HOST -U $DB_USER -D /backups/base/ -P -v

# 3. Archive WAL
cd /var/lib/postgresql/wal/
find . -type f -name "*.wal" -exec gzip {} \;
aws s3 sync . s3://myapp-backups/wal/

# 4. Restore to point in time
# recovery.conf
restore_command = 'cp s3://myapp-backups/wal/%f %p'
recovery_target_time = '2024-01-15 14:30:00'
recovery_target_action = 'promote'
```

---

## Backup Schedule

### Example Schedule

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

## Backup Storage

### 3-2-1 Rule (Golden Standard)

```yaml
storage_strategy:
  rule_3_2_1:
    3_copies: true  # Three copies of data
    2_different_media: true  # Two different storage types
      # 1: Original (production)
      # 2: Backup (disks/HDD)
      # 3: Backup (cloud/offline)
    1_offsite: true  # One offsite copy

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
      storage_class: "glacier_deep_archive"  # For data older than 90 days
      retention: "7 years"  # Compliance requirements
      access_speed: "slow"
      cost: "$0.00099/GB/month"
```

---

## Recovery

### Recovery Strategies

#### Strategy 1: Full Restore (Full + Differential + Incremental)

```bash
#!/bin/bash
# restore_full.sh

RESTORE_DATE="2024-01-15"
RESTORE_DIR="/restore/$RESTORE_DATE"
S3_BUCKET="s3://myapp-backups"

mkdir -p $RESTORE_DIR

echo "🔄 Restoring database to $RESTORE_DATE"

# 1. Download full backup
aws s3 cp $S3_BUCKET/full/full_20240114_020000.sql.gz $RESTORE_DIR/
gunzip $RESTORE_DIR/full_20240114_020000.sql.gz

# 2. Restore full backup
psql -h $DB_HOST -U $DB_USER -d postgres -c "DROP DATABASE IF EXISTS myapp_restore;"
psql -h $DB_HOST -U $DB_USER -d postgres -c "CREATE DATABASE myapp_restore;"

psql -h $DB_HOST -U $DB_USER -d myapp_restore \
  < $RESTORE_DIR/full_20240114_020000.sql

# 3. Download and apply differential backups
for diff in $(aws s3 ls $S3_BUCKET/differential/ --recursive | grep "20240114" | awk '{print $4}'); do
  aws s3 cp $S3_BUCKET/$diff $RESTORE_DIR/
  gunzip $RESTORE_DIR/$(basename $diff)
  psql -h $DB_HOST -U $DB_USER -d myapp_restore \
    < $RESTORE_DIR/$(basename $diff .gz)
done

# 4. Download and apply incremental backups
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

# 1. Stop PostgreSQL
systemctl stop postgresql

# 2. Download base backup
aws s3 cp $S3_BUCKET/base/base_20240114.tar.gz $RESTORE_DIR/
tar -xzf $RESTORE_DIR/base_20240114.tar.gz -C $RESTORE_DIR/

# 3. Configure recovery
cat > $RESTORE_DIR/recovery.conf <<EOF
restore_command = 'cp s3://myapp-backups/wal/%f %p'
recovery_target_time = '$TARGET_TIME'
recovery_target_action = 'promote'
recovery_target_inclusive = false
EOF

# 4. Start PostgreSQL with recovery
systemctl start postgresql

# 5. Monitor logs
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

# 1. Create temporary database for restore
psql -h $DB_HOST -U $DB_USER -d postgres -c "DROP DATABASE IF EXISTS temp_restore;"
psql -h $DB_HOST -U $DB_USER -d postgres -c "CREATE DATABASE temp_restore;"

# 2. Restore full backup to temporary database
aws s3 cp $S3_BUCKET/full/full_20240114_020000.sql.gz $RESTORE_DIR/
gunzip $RESTORE_DIR/full_20240114_020000.sql.gz

psql -h $DB_HOST -U $DB_USER -d temp_restore \
  < $RESTORE_DIR/full_20240114_020000.sql

# 3. Export needed table
pg_dump -h $DB_HOST -U $DB_USER -d temp_restore \
  --table=$TABLE_NAME \
  --data-only \
  > $RESTORE_DIR/table_$TABLE_NAME.sql

# 4. Restore table to production
psql -h $DB_HOST -U $DB_USER -d myapp_production \
  < $RESTORE_DIR/table_$TABLE_NAME.sql

# 5. Clean up temporary database
psql -h $DB_HOST -U $DB_USER -d postgres -c "DROP DATABASE temp_restore;"

echo "✅ Table $TABLE_NAME restored successfully"
```

---

## Backup Verification (Verification)

### Automated Verification

```python
#!/usr/bin/env python3
# verify_backup.py

import subprocess
import psycopg2
import os
from datetime import datetime, timedelta

def verify_latest_backup():
    """Verify latest backup"""

    # Find latest backup
    latest_backup = get_latest_backup()
    print(f"🔍 Verifying backup: {latest_backup}")

    # 1. Check file integrity
    if not verify_file_integrity(latest_backup):
        send_alert("Backup file corrupted", severity="critical")
        return False

    print("✅ File integrity check passed")

    # 2. Restore to test database
    test_db_name = "verify_backup_" + datetime.now().strftime("%Y%m%d_%H%M%S")
    create_test_database(test_db_name)

    try:
        if not restore_to_test(latest_backup, test_db_name):
            send_alert("Restore verification failed", severity="critical")
            return False

        print("✅ Restore test passed")

        # 3. Run basic tests
        if not run_basic_tests(test_db_name):
            send_alert("Backup data validation failed", severity="critical")
            return False

        print("✅ Data validation tests passed")

        # 4. Check freshness
        backup_age = get_backup_age(latest_backup)
        if backup_age > timedelta(hours=25):
            send_alert(f"Backup is too old: {backup_age}", severity="warning")

        print(f"✅ Backup age: {backup_age} (good)")

        send_alert("Backup verification successful", severity="info")
        return True

    finally:
        cleanup_test_database(test_db_name)

def get_latest_backup():
    """Find latest backup"""
    result = subprocess.run(
        ["aws", "s3", "ls", "s3://myapp-backups/full/", "--recursive"],
        capture_output=True,
        text=True
    )

    files = result.stdout.strip().split('\n')
    latest = sorted(files, reverse=True)[0]
    return latest.split()[-1]

def verify_file_integrity(backup_path):
    """Check file integrity"""
    # Download file
    local_path = f"/tmp/{os.path.basename(backup_path)}"
    subprocess.run(["aws", "s3", "cp", f"s3://myapp-backups/{backup_path}", local_path])

    # Check checksum
    result = subprocess.run(["md5sum", local_path], capture_output=True, text=True)
    md5 = result.stdout.split()[0]

    # Compare with previously saved checksum
    with open(f"{local_path}.md5", "r") as f:
        expected_md5 = f.read().strip()

    os.remove(local_path)
    return md5 == expected_md5

def restore_to_test(backup_path, test_db_name):
    """Restore to test database"""
    local_path = f"/tmp/{os.path.basename(backup_path)}"

    # Download and unzip
    subprocess.run(["aws", "s3", "cp", f"s3://myapp-backups/{backup_path}", local_path])
    subprocess.run(["gunzip", local_path])

    # Restore
    result = subprocess.run(
        ["pg_restore", "-h", os.getenv("DB_HOST"), "-U", os.getenv("DB_USER"), "-d", test_db_name, local_path[:-3]],
        capture_output=True
    )

    os.remove(local_path[:-3])  # .dump file
    return result.returncode == 0

def run_basic_tests(db_name):
    """Run basic tests on restored data"""
    conn = psycopg2.connect(
        host=os.getenv("DB_HOST"),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
        database=db_name
    )

    cursor = conn.cursor()

    try:
        # Test 1: Check connection
        cursor.execute("SELECT 1")
        assert cursor.fetchone()[0] == 1

        # Test 2: Check user count
        cursor.execute("SELECT COUNT(*) FROM users")
        user_count = cursor.fetchone()[0]
        assert user_count > 0, "No users found"

        # Test 3: Check indexes
        cursor.execute("""
            SELECT tablename, indexname
            FROM pg_indexes
            WHERE schemaname = 'public'
        """)
        indexes = cursor.fetchall()
        assert len(indexes) > 0, "No indexes found"

        # Test 4: Check foreign keys
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
    """Get backup creation time"""
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
    """Create test database"""
    subprocess.run([
        "psql", "-h", os.getenv("DB_HOST"), "-U", os.getenv("DB_USER"), "-d", "postgres",
        "-c", f"CREATE DATABASE {db_name};"
    ], check=True)

def cleanup_test_database(db_name):
    """Clean up test database"""
    subprocess.run([
        "psql", "-h", os.getenv("DB_HOST"), "-U", os.getenv("DB_USER"), "-d", "postgres",
        "-c", f"DROP DATABASE IF EXISTS {db_name};"
    ], check=True)

def send_alert(message, severity):
    """Send alert"""
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

## Post-Incident Recovery

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

## Storage Costs

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

## Automation

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

## Configuration

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

### Storage
- ✅ 3-2-1 rule (3 copies, 2 media, 1 offsite)
- ✅ Verify backups automatically every day
- ✅ Test restore process monthly
- ✅ Use encryption (at rest and in transit)
- ✅ Regularly rotate encryption keys
- ✅ Keep checksums (md5/sha256)

### Recovery
- ✅ Document all recovery procedures
- ✅ Practice disaster recovery quarterly
- ✅ Measure RTO and RPO
- ✅ Keep runbooks up-to-date
- ✅ Have clear ownership

### Security
- ✅ RBAC for backup access
- ✅ Audit logs of all operations
- ✅ MFA for storage access
- ✅ Network isolation for backup servers
- ✅ Regular penetration testing

### Processes
- ✅ Review and test disaster recovery plan quarterly
- ✅ Update runbooks after each incident
- ✅ Train new team members on recovery procedures
- ✅ Conduct blameless postmortems for failures
- ✅ Automate where possible

---

## Tools

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

*Backup & Recovery - data backup and recovery practice*
