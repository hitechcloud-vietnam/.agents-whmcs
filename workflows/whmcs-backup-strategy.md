# WHMCS Backup Strategy Workflow

## Purpose

Implement a comprehensive backup strategy for WHMCS to protect against data loss, corruption, and disasters. This workflow covers backup types, scheduling, verification, restoration testing, and offsite storage.

## Prerequisites

- WHMCS installation (version 8.x)
- Sufficient storage for backups
- Access to database and file system
- Offsite storage location (S3, Azure, Google Cloud)
- Monitoring system for backup alerts

## Workflow Steps

### Step 1: Backup Architecture Design

Design a multi-tier backup strategy:

```
┌─────────────────────────────────────────────────────────────┐
│                    Backup Architecture                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   Hourly     │    │   Daily      │    │   Weekly     │  │
│  │   (MySQL)    │    │   (Full)     │    │   (Archive)  │  │
│  │              │    │              │    │              │  │
│  │  Retention:  │    │  Retention:  │    │  Retention:  │
│  │  24 hours    │    │  30 days     │    │  90 days     │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘  │
│         │                   │                   │           │
│         └───────────────────┼───────────────────┘           │
│                             │                               │
│                    ┌────────▼────────┐                    │
│                    │   Local Storage  │                    │
│                    │   (Fast Restore) │                    │
│                    └────────┬────────┘                    │
│                             │                               │
│                    ┌────────▼────────┐                    │
│                    │   Offsite S3    │                    │
│                    │   (Disaster)    │                    │
│                    └─────────────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

### Step 2: Database Backup Script

```bash
#!/bin/bash
# /opt/scripts/backup_database.sh

set -euo pipefail

# Configuration
DB_HOST="localhost"
DB_NAME="whmcs_main"
DB_USER="whmcs"
DB_PASS="password"
BACKUP_ROOT="/var/backups/whmcs"
RETENTION_DAYS=30
S3_BUCKET="s3://company-whmcs-backups"
S3_REGION="us-east-1"
LOG_FILE="/var/log/backup/database.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

cleanup() {
    log "Cleaning up..."
    # Remove old local backups
    find "$BACKUP_ROOT" -type d -mtime +$RETENTION_DAYS -exec rm -rf {} \;
}

# Start backup
log "Starting database backup for $DB_NAME"

# Create backup directory
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="$BACKUP_ROOT/$TIMESTAMP"
mkdir -p "$BACKUP_DIR"

# Lock table and backup
log "Locking tables and creating backup..."
mysqldump -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASS" \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --hex-blob \
    --master-data=2 \
    --flush-logs \
    --log-error="$BACKUP_DIR/backup_error.log" \
    "$DB_NAME" | gzip > "$BACKUP_DIR/${DB_NAME}.sql.gz"

# Verify backup
BACKUP_SIZE=$(du -h "$BACKUP_DIR/${DB_NAME}.sql.gz" | cut -f1)
ROW_COUNT=$(zcat "$BACKUP_DIR/${DB_NAME}.sql.gz" | grep -c "INSERT INTO" || echo "0")

log "Database backup complete. Size: $BACKUP_SIZE, Tables processed: $ROW_COUNT"

# Generate checksum
cd "$BACKUP_DIR"
sha256sum "${DB_NAME}.sql.gz" > "${DB_NAME}.sql.gz.sha256"

# Upload to S3
log "Uploading to S3..."
aws s3 cp "$BACKUP_DIR/" "$S3_BUCKET/database/$TIMESTAMP/" \
    --recursive \
    --storage-class STANDARD_IA \
    --region "$S3_REGION"

# Cleanup old backups
cleanup

log "Database backup completed successfully"
```

### Step 3: File System Backup Script

```bash
#!/bin/bash
# /opt/scripts/backup_files.sh

set -euo pipefail

# Configuration
WHXCS_ROOT="/var/www/whmcs"
BACKUP_ROOT="/var/backups/whmcs/files"
S3_BUCKET="s3://company-whmcs-backups"
EXCLUDE_PATTERN="--exclude='cache/*' \
                  --exclude='storage/templates_c/*' \
                  --exclude='storage/logs/*' \
                  --exclude='*.log' \
                  --exclude='node_modules/*' \
                  --exclude='vendor/*'"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log "Starting WHMCS file backup"

# Create backup directory
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="$BACKUP_ROOT/$TIMESTAMP"
mkdir -p "$BACKUP_DIR"

# Create archive (excluding cache and logs)
log "Creating file archive..."
tar -czf "$BACKUP_DIR/whmcs_files.tar.gz" \
    -C "$(dirname $WHXCS_ROOT)" \
    $(echo $EXCLUDE_PATTERN | sed 's/--exclude=/-X /g') \
    "$(basename $WHXCS_ROOT)"

# Backup configuration separately (encrypted)
log "Backing up configuration..."
tar -czf "$BACKUP_DIR/whmcs_config.tar.gz" \
    -C "$WHXCS_ROOT" \
    configuration.php

# Encrypt configuration backup
openssl enc -aes-256-cbc -salt -pbkdf2 \
    -in "$BACKUP_DIR/whmcs_config.tar.gz" \
    -out "$BACKUP_DIR/whmcs_config.tar.gz.enc" \
    -pass pass:"$ENCRYPTION_PASSWORD"
rm "$BACKUP_DIR/whmcs_config.tar.gz"

# Generate checksums
cd "$BACKUP_DIR"
sha256sum *.tar.gz* > checksums.sha256

# Upload to S3
log "Uploading to S3..."
aws s3 cp "$BACKUP_DIR/" "$S3_BUCKET/files/$TIMESTAMP/" \
    --recursive \
    --storage-class STANDARD_IA \
    --region "$S3_REGION"

# Cleanup
find "$BACKUP_ROOT" -type d -mtime +30 -exec rm -rf {} \;

log "File backup completed successfully"
```

### Step 4: Incremental Backup with RSYNC

```bash
#!/bin/bash
# /opt/scripts/incremental_backup.sh
# For real-time or frequent incremental backups

WHXCS_ROOT="/var/www/whmcs"
BACKUP_ROOT="/var/backups/whmcs/incremental"
REMOTE_USER="backup"
REMOTE_HOST="backup-server"
REMOTE_PATH="/backup/whmcs"
EXCLUDES="--exclude=cache/* --exclude=storage/templates_c/* --exclude=*.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

# Create snapshot directory
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
SNAPSHOT_DIR="$BACKUP_ROOT/snapshots/$TIMESTAMP"

# Use rsync with link-dest for efficient incremental backups
rsync -avz --delete \
    $EXCLUDES \
    --link-dest="$BACKUP_ROOT/current" \
    "$WHXCS_ROOT/" \
    "$SNAPSHOT_DIR/"

# Update current link
rm -rf "$BACKUP_ROOT/current"
ln -s "$SNAPSHOT_DIR" "$BACKUP_ROOT/current"

log "Incremental backup complete: $TIMESTAMP"
```

### Step 5: Backup Verification Script

```bash
#!/bin/bash
# /opt/scripts/verify_backup.sh
# Verify backup integrity after each backup

set -euo pipefail

BACKUP_ROOT="/var/backups/whmcs"
TEST_DB="whmcs_backup_test"
LOG_FILE="/var/log/backup/verify.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "Starting backup verification"

# Get latest backup
LATEST=$(ls -td "$BACKUP_ROOT"/*/ | head -1)
log "Testing backup: $LATEST"

# Verify checksums
if [ -f "$LATEST/checksums.sha256" ]; then
    if sha256sum -c "$LATEST/checksums.sha256"; then
        log "Checksum verification passed"
    else
        log "ERROR: Checksum verification failed!"
        exit 1
    fi
fi

# Extract and test database
log "Testing database restore..."
gunzip < "$LATEST"/*.sql.gz | head -100 > /dev/null
log "Database backup readable"

# Test database restore to test server
mysql -u root -p -e "DROP DATABASE IF EXISTS $TEST_DB"
mysql -u root -p -e "CREATE DATABASE $TEST_DB"
gunzip < "$LATEST"/*.sql.gz | mysql -u root -p "$TEST_DB"

# Verify critical tables
TABLES=("tblclients" "tblhosting" "tblinvoices" "tblorders" "tblproducts")
for table in "${TABLES[@]}"; do
    COUNT=$(mysql -u root -p -N -e "SELECT COUNT(*) FROM $TEST_DB.$table" 2>/dev/null || echo "0")
    if [ "$COUNT" = "0" ]; then
        log "WARNING: $table is empty or missing"
    else
        log "$table: OK ($COUNT records)"
    fi
done

# Cleanup test database
mysql -u root -p -e "DROP DATABASE $TEST_DB"

# Verify file archive
log "Testing file archive..."
tar -tzf "$LATEST"/*.tar.gz | head -20 > /dev/null
log "File archive readable"

log "Backup verification completed successfully"
```

### Step 6: WHMCS Backup Module Configuration

```php
<?php
// /var/www/whmcs/includes/hooks/backup_hook.php
// Automated WHMCS backup hook

add_hook('DailyCronJob', 1, function($vars) {
    $backupPath = '/var/backups/whmcs';
    $timestamp = date('Y-m-d_H-i-s');
    
    // Log backup start
    logActivity("Automated backup started: $timestamp");
    
    // Get database connection
    $pdo = new PDO('mysql:host=localhost', 'whmcs', 'password');
    
    // Create backup metadata
    $metadata = [
        'timestamp' => $timestamp,
        'version' => \App::getVersion(),
        'php_version' => PHP_VERSION,
        'tables' => []
    ];
    
    // Get table statistics
    $tables = $pdo->query("SHOW TABLES")->fetchAll(PDO::FETCH_COLUMN);
    foreach ($tables as $table) {
        $count = $pdo->query("SELECT COUNT(*) FROM `$table`")->fetchColumn();
        $metadata['tables'][$table] = $count;
    }
    
    // Save metadata
    file_put_contents(
        "$backupPath/$timestamp/metadata.json",
        json_encode($metadata, JSON_PRETTY_PRINT)
    );
    
    logActivity("Automated backup completed: $timestamp");
    
    // Check backup size and alert if abnormal
    $backupSize = exec("du -sh $backupPath/$timestamp | cut -f1");
    logActivity("Backup size: $backupSize");
});
```

### Step 7: Backup Rotation Schedule

```bash
# /etc/cron.d/whmcs-backup
# Backup rotation schedule

# Hourly database backup (for point-in-time recovery)
0 * * * * root /opt/scripts/backup_database_hourly.sh >> /var/log/backup/hourly.log 2>&1

# Daily full backup (1:00 AM)
0 1 * * * root /opt/scripts/backup_full.sh >> /var/log/backup/daily.log 2>&1

# Weekly archive backup (Sunday 2:00 AM)
0 2 * * 0 root /opt/scripts/backup_archive.sh >> /var/log/backup/weekly.log 2>&1

# Verify backups daily (6:00 AM)
0 6 * * * root /opt/scripts/verify_backup.sh >> /var/log/backup/verify.log 2>&1

# Cleanup old backups (daily at 3:00 AM)
0 3 * * * root /opt/scripts/cleanup_backups.sh >> /var/log/backup/cleanup.log 2>&1

# Monitor backup status (every 6 hours)
0 */6 * * * root /opt/scripts/check_backup_status.sh
```

### Step 8: Restore Procedure

```bash
#!/bin/bash
# /opt/scripts/restore_whmcs.sh
# Full restore from backup

set -euo pipefail

BACKUP_DATE=${1:-latest}
WHXCS_ROOT="/var/www/whmcs"
DB_NAME="whmcs_main"
DB_USER="whmcs"
DB_PASS="password"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log "Starting WHMCS restore from $BACKUP_DATE"

# Determine backup source
if [ "$BACKUP_DATE" = "latest" ]; then
    BACKUP_PATH=$(ls -td /var/backups/whmcs/*/ | head -1)
else
    BACKUP_PATH="/var/backups/whmcs/$BACKUP_DATE"
fi

if [ ! -d "$BACKUP_PATH" ]; then
    log "ERROR: Backup path not found: $BACKUP_PATH"
    exit 1
fi

log "Using backup: $BACKUP_PATH"

# 1. Put WHMCS in maintenance mode
touch "$WHXCS_ROOT/.maintenance"

# 2. Stop cron jobs
systemctl stop cron 2>/dev/null || true

# 3. Backup current state (just in case)
CURRENT_BACKUP="/tmp/whmcs_current_$(date +%Y%m%d_%H%M%S)"
log "Backing up current state to $CURRENT_BACKUP"
cp -r "$WHXCS_ROOT" "$CURRENT_BACKUP"

# 4. Restore files
log "Restoring files..."
tar -xzf "$BACKUP_PATH/whmcs_files.tar.gz" -C "$(dirname $WHXCS_ROOT)"
tar -xzf "$BACKUP_PATH/whmcs_config.tar.gz.enc" -C /tmp

# Decrypt and restore configuration
openssl enc -aes-256-cbc -d -pbkdf2 \
    -in "$BACKUP_PATH/whmcs_config.tar.gz.enc" \
    -out /tmp/whmcs_config.tar.gz \
    -pass pass:"$ENCRYPTION_PASSWORD"
tar -xzf /tmp/whmcs_config.tar.gz -C "$WHXCS_ROOT"
chown www-data:www-data "$WHXCS_ROOT/configuration.php"
chmod 600 "$WHXCS_ROOT/configuration.php"

# 5. Restore database
log "Restoring database..."
mysql -u root -p -e "DROP DATABASE IF EXISTS $DB_NAME"
mysql -u root -p -e "CREATE DATABASE $DB_NAME"
gunzip < "$BACKUP_PATH/whmcs_main.sql.gz" | mysql -u root -p "$DB_NAME"

# 6. Fix permissions
log "Fixing permissions..."
chown -R www-data:www-data "$WHXCS_ROOT"
find "$WHXCS_ROOT" -type d -exec chmod 755 {} \;
find "$WHXCS_ROOT" -type f -exec chmod 644 {} \;
chmod 600 "$WHXCS_ROOT/configuration.php"

# 7. Clear cache
log "Clearing cache..."
php "$WHXCS_ROOT/crons/cron.php" a=clearCache

# 8. Verify
log "Verifying restore..."
curl -f http://localhost/ | grep -q "WHMCS" && log "Restore successful!" || log "WARNING: Verification failed"

# 9. Remove maintenance mode
rm "$WHXCS_ROOT/.maintenance"

# 10. Restart services
systemctl start cron 2>/dev/null || true

log "Restore completed successfully"
```

## Backup Retention Policy

| Backup Type | Retention | Storage | Purpose |
|-------------|-----------|---------|---------|
| Hourly DB | 24 hours | Local | Point-in-time recovery |
| Daily Full | 30 days | Local + S3 | Disaster recovery |
| Weekly Archive | 90 days | S3 Glacier | Long-term retention |
| Monthly Archive | 1 year | S3 Glacier | Compliance |
| Yearly Archive | 7 years | S3 Glacier Deep | Legal/compliance |

## Best Practices

1. **Follow 3-2-1 Rule**: 3 copies, 2 media types, 1 offsite
2. **Test Restores**: Verify backups can actually be restored
3. **Encrypt Backups**: Especially for sensitive data
4. **Monitor Backups**: Alert on failures immediately
5. **Document Procedures**: Written restore procedures
6. **Version Control**: Track backup versions and changes

## Common Pitfalls

- **Not Testing Backups**: Backups may be corrupt
- **Insufficient Storage**: Backups fail due to space
- **Incomplete Backups**: Missing critical tables
- **Single Point of Failure**: Only one backup copy
- **Long Retention**: Old backups waste space

## Verification Checklist

- [ ] All backup jobs running on schedule
- [ ] Backup verification tests passing
- [ ] Offsite backups completing
- [ ] Backup size monitoring active
- [ ] Restore procedures documented
- [ ] Restore tested quarterly
- [ ] Encryption configured
- [ ] Alerts configured for failures

## Related Documentation

- [WHMCS Disaster Recovery](whmcs-disaster-recovery-workflow.md)
- [WHMCS Database Migration](whmcs-database-migration-workflow.md)
- [WHMCS Multi-Server Workflow](whmcs-multi-server-workflow.md)