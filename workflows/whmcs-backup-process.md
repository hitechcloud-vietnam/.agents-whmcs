# WHMCS Backup Procedure Workflow

## Purpose
Create comprehensive backup of WHMCS installation

## Prerequisites
- WHMCS installed
- SSH access
- Sufficient storage space

## Step 1: Prepare Backup Location

```bash
# Create backup directory
mkdir -p /backup/whmcs/$(date +%Y%m%d)

# Verify disk space
df -h /backup
```

## Step 2: Database Backup

```bash
# Single database backup
mysqldump -u root -p whmcs_db > /backup/whmcs/$(date +%Y%m%d)/whmcs_db.sql

# Backup with compression
mysqldump -u root -p whmcs_db | gzip > /backup/whmcs/$(date +%Y%m%d)/whmcs_db.sql.gz

# Backup all databases
mysqldump -u root -p --all-databases | gzip > /backup/whmcs/$(date +%Y%m%d)/all_databases.sql.gz
```

## Step 3: Configuration Backup

```bash
# Backup configuration file
cp /var/www/whmcs/configuration.php /backup/whmcs/$(date +%Y%m%d)/

# Backup .htaccess
cp /var/www/whmcs/.htaccess /backup/whmcs/$(date +%Y%m%d)/ 2>/dev/null || true

# Backup custom config
cp /var/www/whmcs/custom/config.php /backup/whmcs/$(date +%Y%m%d)/ 2>/dev/null || true
```

## Step 4: Files Backup

```bash
cd /var/www/whmcs

# Backup entire installation (excluding cache)
tar -czf /backup/whmcs/$(date +%Y%m%d)/whmcs_files.tar.gz \
  --exclude='./templates_c/*' \
  --exclude='./cache/*' \
  --exclude='./downloads/*' \
  .

# OR backup everything
tar -czf /backup/whmcs/$(date +%Y%m%d)/whmcs_complete.tar.gz .
```

## Step 5: Email Templates Backup

```bash
tar -czf /backup/whmcs/$(date +%Y%m%d)/email_templates.tar.gz \
  /var/www/whmcs/templates/*/email_*.tpl \
  2>/dev/null || true
```

## Step 6: Module Backup

```bash
tar -czf /backup/whmcs/$(date +%Y%m%d)/modules.tar.gz \
  /var/www/whmcs/modules/
```

## Step 7: Verify Backup Integrity

```bash
# Check database backup
gunzip -t /backup/whmcs/$(date +%Y%m%d)/whmcs_db.sql.gz

# Check files backup
tar -tzf /backup/whmcs/$(date +%Y%m%d)/whmcs_files.tar.gz | head

# Get backup sizes
ls -lh /backup/whmcs/$(date +%Y%m%d)/
```

## Step 8: Off-Site Backup

```bash
# Copy to remote server
rsync -avz /backup/whmcs/$(date +%Y%m%d)/ user@backupserver:/backups/whmcs/

# Copy to cloud storage
# Using rclone
rclone copy /backup/whmcs/$(date +%Y%m%d)/ gdrive:whmcs-backups/$(date +%Y%m%d)/

# Using scp
scp -r /backup/whmcs/$(date +%Y%m%d)/ user@remoteserver:/backups/
```

## Step 9: Automated Backup Script

Create script at `/usr/local/bin/whmcs-backup.sh`:

```bash
#!/bin/bash
# WHMCS Automated Backup Script

DATE=$(date +%Y%m%d)
BACKUP_DIR="/backup/whmcs/$DATE"
WHMCS_DIR="/var/www/whmcs"
DB_NAME="whmcs_db"
DB_USER="root"
DB_PASS="password"

# Create backup directory
mkdir -p $BACKUP_DIR

# Database backup
mysqldump -u $DB_USER -p$DB_PASS $DB_NAME | gzip > $BACKUP_DIR/whmcs_db.sql.gz

# Files backup
tar -czf $BACKUP_DIR/whmcs_files.tar.gz \
  --exclude='./templates_c/*' \
  --exclude='./cache/*' \
  -C $WHMCS_DIR .

# Copy configuration
cp $WHMCS_DIR/configuration.php $BACKUP_DIR/

# Clean up backups older than 30 days
find /backup/whmcs -type d -mtime +30 -exec rm -rf {} \;

# Sync to remote
rclone copy $BACKUP_DIR gdrive:whmcs-backups/$DATE/

echo "Backup completed: $DATE"
```

```bash
chmod +x /usr/local/bin/whmcs-backup.sh
```

## Step 10: Schedule Backup Cron

```bash
crontab -e
```

Add:
```cron
# WHMCS Daily Backup at 2 AM
0 2 * * * /usr/local/bin/whmcs-backup.sh >> /var/log/whmcs-backup.log 2>&1
```

## Step 11: Backup Verification Checklist

After each backup:
- [ ] Database backup exists and is valid
- [ ] Files backup exists and is valid
- [ ] Configuration backup exists
- [ ] Backup size is reasonable
- [ ] Off-site copy completed

## Backup Checklist

- [ ] Backup directory prepared
- [ ] Database backed up
- [ ] Configuration backed up
- [ ] Files backed up
- [ ] Email templates backed up
- [ ] Modules backed up
- [ ] Backup verified
- [ ] Off-site backup copied
- [ ] Automated script created
- [ ] Cron scheduled
