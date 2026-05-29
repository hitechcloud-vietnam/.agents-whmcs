# WHMCS Backup Setup Workflow

## Description
Comprehensive backup configuration for WHMCS including automated backups.

## Prerequisites
- SSH access to WHMCS server
- Sufficient storage for backups
- Remote backup destination (recommended)
- Root/sudo access

## Steps

### Step 1: Create Backup Directory Structure
```bash
# Local backup directory
mkdir -p /backups/whmcs/{daily,weekly,monthly,database,config,logs}

# Set permissions
chmod 750 /backups/whmcs
chmod 750 /backups/whmcs/*
```

### Step 2: Create Backup Script
```bash
cat > /usr/local/bin/whmcs-backup.sh << 'EOF'
#!/bin/bash

# Configuration
BACKUP_DIR="/backups/whmcs"
WHMCS_DIR="/var/www/whmcs"
DB_NAME="whmcs"
DB_USER="whmcs"
DB_PASS="your_password"
REMOTE_HOST="backup-server"
REMOTE_USER="backup"
REMOTE_DIR="/backups/whmcs"

# Date stamp
DATE=$(date +%Y%m%d_%H%M%S)

# Logging
exec > >(tee -a /var/log/whmcs-backup.log) 2>&1

echo "=== WHMCS Backup Started: $DATE ==="

# Create temporary directory
TEMP_DIR="/tmp/whmcs_backup_$DATE"
mkdir -p "$TEMP_DIR"

# 1. Database backup
echo "Backing up database..."
mysqldump -u"$DB_USER" -p"$DB_PASS" "$DB_NAME" | gzip > "$TEMP_DIR/database.sql.gz"
if [ $? -eq 0 ]; then
    echo "Database backup completed"
else
    echo "Database backup failed!"
    exit 1
fi

# 2. Files backup
echo "Backing up files..."
tar -czpf "$TEMP_DIR/files.tar.gz" -C "$(dirname $WHMCS_DIR)" "$(basename $WHMCS_DIR)" \
    --exclude="$WHMCS_DIR/templates_c/*" \
    --exclude="$WHMCS_DIR/cache/*"

# 3. Configuration backup
echo "Backing up configuration..."
cp "$WHMCS_DIR/configuration.php" "$TEMP_DIR/"

# 4. Create manifest
cat > "$TEMP_DIR/manifest.txt" << MANIFEST
Backup Date: $DATE
WHMCS Version: $(grep '^\$myversion' $WHMCS_DIR/init.php | cut -d"'" -f2)
Database: $DB_NAME
Files Location: $WHMCS_DIR
MANIFEST

# 5. Create archive
cd "$TEMP_DIR"
tar -czf "$BACKUP_DIR/daily/whmcs_backup_$DATE.tar.gz" *

# 6. Upload to remote (if configured)
if [ -n "$REMOTE_HOST" ]; then
    echo "Uploading to remote server..."
    rsync -avz "$BACKUP_DIR/daily/whmcs_backup_$DATE.tar.gz" \
        "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/daily/"
fi

# 7. Cleanup old backups
find "$BACKUP_DIR/daily" -name "whmcs_backup_*.tar.gz" -mtime +7 -delete
find "$BACKUP_DIR/weekly" -name "whmcs_backup_*.tar.gz" -mtime +30 -delete
find "$BACKUP_DIR/monthly" -name "whmcs_backup_*.tar.gz" -mtime +365 -delete

# 8. Final cleanup
rm -rf "$TEMP_DIR"

echo "=== WHMCS Backup Completed: $DATE ==="
EOF

chmod +x /usr/local/bin/whmcs-backup.sh
```

### Step 3: Setup Cron Jobs
```bash
# Edit crontab
crontab -e

# Add these entries:
# Daily backup at 2 AM
0 2 * * * /usr/local/bin/whmcs-backup.sh >> /var/log/whmcs-backup-cron.log 2>&1

# Weekly backup on Sunday at 3 AM
0 3 * * 0 cp /backups/whmcs/daily/$(date +\%Y\%m\%d)* /backups/whmcs/weekly/

# Monthly backup on 1st at 4 AM
0 4 1 * * cp /backups/whmcs/daily/$(date +\%Y\%m\%d)* /backups/whmcs/monthly/
```

### Step 4: Setup Remote Backup (Optional)
```bash
# Setup SSH key for passwordless transfer
ssh-keygen -t rsa -b 4096
ssh-copy-id backup@remote-server

# Test connection
ssh backup@remote-server "ls /backups/whmcs"
```

### Step 5: Verify Backup Integrity
```bash
# Create verification script
cat > /usr/local/bin/verify-backup.sh << 'EOF'
#!/bin/bash

BACKUP_FILE="$1"
TEMP_DIR="/tmp/backup_verify_$$"

mkdir -p "$TEMP_DIR"
tar -xzf "$BACKUP_FILE" -C "$TEMP_DIR"

# Check database
if gunzip -t "$TEMP_DIR/database.sql.gz"; then
    echo "Database backup: OK"
else
    echo "Database backup: CORRUPTED"
    exit 1
fi

# Check files
if [ -d "$TEMP_DIR/whmcs" ]; then
    echo "Files backup: OK"
else
    echo "Files backup: MISSING"
    exit 1
fi

# Check configuration
if [ -f "$TEMP_DIR/configuration.php" ]; then
    echo "Configuration: OK"
else
    echo "Configuration: MISSING"
    exit 1
fi

rm -rf "$TEMP_DIR"
echo "Backup verification passed!"
EOF

chmod +x /usr/local/bin/verify-backup.sh
```

### Step 6: Setup Monitoring
```bash
# Add to backup script for monitoring
if [ $? -eq 0 ]; then
    echo "Backup successful" | mail -s "WHMCS Backup Success" admin@example.com
else
    echo "Backup failed" | mail -s "WHMCS Backup FAILED" admin@example.com
fi
```

## Backup Retention Policy

| Type | Frequency | Retention | Storage |
|------|-----------|-----------|---------|
| Daily | Every day 2 AM | 7 days | Local + Remote |
| Weekly | Every Sunday 3 AM | 30 days | Local + Remote |
| Monthly | 1st of month 4 AM | 365 days | Remote only |

## Backup Contents Checklist
- [ ] Database (complete)
- [ ] configuration.php
- [ ] attachments/
- [ ] downloads/
- [ ] language files
- [ ] Custom modules
- [ ] Custom templates
- [ ] SSL certificates
- [ ] Cron logs

## Tags
- backup
- disaster-recovery
- automation
- cron