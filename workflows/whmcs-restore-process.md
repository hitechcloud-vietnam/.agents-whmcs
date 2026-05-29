# WHMCS Restore Process Workflow

## Description
Step-by-step guide for restoring WHMCS from backup.

## Prerequisites
- Valid backup files (database + files)
- SSH access to server
- Same or compatible software versions
- Root access

## Warning
Always verify backup integrity before restoring.

## Steps

### Step 1: Verify Backup Files
```bash
# List backup files
ls -lh /backups/whmcs/daily/

# Verify backup integrity
tar -tzf /backups/whmcs/daily/whmcs_backup_*.tar.gz > /dev/null
echo "Backup archive is valid"

# Check database backup
gunzip -t /backups/whmcs/daily/database.sql.gz
echo "Database backup is valid"
```

### Step 2: Enable Maintenance Mode
1. Login to WHMCS Admin
2. Go to Configuration > System Settings > General
3. Enable Maintenance Mode
4. Set custom message: "System restore in progress"

### Step 3: Stop Services
```bash
# Stop web server
systemctl stop nginx  # or systemctl stop apache2

# Stop cron jobs
systemctl stop cron

# Ensure MySQL is running
systemctl start mysql
```

### Step 4: Create Emergency Backup
```bash
# Backup current state before restore
cd /var/www
tar -czvf whmcs_pre_restore_$(date +%Y%m%d).tar.gz whmcs/
mysqldump -u root -p whmcs > whmcs_pre_restore_db_$(date +%Y%m%d).sql
```

### Step 5: Extract Backup Files
```bash
# Create temp directory
mkdir -p /tmp/whmcs_restore
cd /tmp/whmcs_restore

# Extract backup
tar -xzvf /backups/whmcs/daily/whmcs_backup_*.tar.gz

# List contents
ls -la
```

### Step 6: Restore Database
```bash
# Drop existing database
mysql -u root -p -e "DROP DATABASE IF EXISTS whmcs;"
mysql -u root -p -e "CREATE DATABASE whmcs CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# Restore database
gunzip < database.sql.gz | mysql -u root -p whmcs

# Verify restore
mysql -u root -p -e "SELECT COUNT(*) FROM whmcs.tblclients;"
```

### Step 7: Restore Files
```bash
# Remove current installation
rm -rf /var/www/whmcs

# Restore files
# (Note: adjust path based on backup structure)
cp -r /path/to/extracted/whmcs /var/www/

# Restore custom files if needed
cp -r /path/to/attachments /var/www/whmcs/
```

### Step 8: Restore Configuration
```bash
# Copy configuration file
cp /path/to/backup/configuration.php /var/www/whmcs/

# Update database credentials if needed
# Edit configuration.php with correct credentials
```

### Step 9: Set Permissions
```bash
cd /var/www/whmcs

# Set ownership
chown -R www-data:www-data .

# Set base permissions
chmod 755 .
chmod 644 configuration.php

# Set directory permissions
chmod 755 templates_c attachments downloads language

# Set file permissions
find . -type f -exec chmod 644 {} \;
find ./templates_c -type f -exec chmod 644 {} \;
find ./attachments -type f -exec chmod 644 {} \;
```

### Step 10: Clear Cache
```bash
cd /var/www/whmcs

# Remove cached files
rm -rf templates_c/*
rm -rf cache/*

# Run cache clear CLI
php -q whmcs/cli/clear-cache.php
```

### Step 11: Restart Services
```bash
# Start MySQL
systemctl start mysql

# Start web server
systemctl start nginx  # or systemctl start apache2

# Start cron
systemctl start cron
```

### Step 12: Verify Restore
1. Login to WHMCS Admin
2. Check System Health
3. Verify client data
4. Check invoices
5. Test payment processing
6. Verify email sending
7. Check active services

### Step 13: Disable Maintenance Mode
1. Go to Configuration > System Settings > General
2. Disable Maintenance Mode
3. Clear any custom message

## Point-in-Time Recovery (MySQL)

```bash
# For point-in-time recovery using binary logs
# Enable binary logging in MySQL first

# List binary logs
mysql -u root -p -e "SHOW BINARY LOGS;"

# Restore to specific point in time
mysqlbinlog --stop-datetime="2024-01-15 15:00:00" /var/lib/mysql/mysql-bin.* | mysql -u root -p whmcs
```

## Partial Restore Options

### Restore Only Database
```bash
mysql -u root -p whmcs < /path/to/backup/database.sql
```

### Restore Only Files
```bash
rsync -avz /path/to/backup/files/ /var/www/whmcs/
```

### Restore Single Table
```bash
# Extract specific table
mysqldump -u root -p whmcs tblinvoices > tblinvoices.sql
mysql -u root -p whmcs < tblinvoices.sql
```

## Verification Checklist
- [ ] Admin login works
- [ ] Client data intact
- [ ] Invoices accessible
- [ ] Products/Services listed
- [ ] Payment gateways configured
- [ ] Email sending functional
- [ ] Cron jobs running

## Rollback Procedure
```bash
# If restore failed, rollback to pre-restore backup
systemctl stop nginx
rm -rf /var/www/whmcs
tar -xzvf /var/www/whmcs_pre_restore_*.tar.gz -C /var/www/
mysql -u root -p -e "DROP DATABASE IF EXISTS whmcs;"
mysql -u root -p whmcs < /var/www/whmcs_pre_restore_db_*.sql
systemctl start nginx
```

## Tags
- restore
- backup
- disaster-recovery
- recovery