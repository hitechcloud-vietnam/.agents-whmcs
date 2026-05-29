# WHMCS System Upgrade Workflow

## Description
Step-by-step guide for upgrading WHMCS to a newer version.

## Prerequisites
- WHMCS current version information
- Full backup of current installation
- SSH access to server
- Maintenance mode capability

## Steps

### Step 1: Check Current Version
```bash
# Check current WHMCS version
grep "^\$mybr与应用" /var/www/whmcs/init.php
# Or check admin dashboard
```

### Step 2: Create Full Backup
```bash
# Backup entire WHMCS directory
cd /var/www
cp -r whmcs whmcs_backup_pre_upgrade_$(date +%Y%m%d)

# Backup database
mysqldump -u root -p whmcs > whmcs_db_backup_$(date +%Y%m%d).sql

# Backup configuration
cp /var/www/whmcs/configuration.php /tmp/configuration_backup.php
```

### Step 3: Enable Maintenance Mode
1. Login to WHMCS Admin
2. Go to Configuration > System Settings > General
3. Enable Maintenance Mode
4. Add custom message

### Step 4: Download New Version
```bash
# Download latest WHMCS
cd /tmp
wget https://download.whmcs.com/whmcs.zip
unzip -o whmcs.zip
```

### Step 5: Prepare Update Files
```bash
# Copy new files over existing installation
cp -rf /tmp/whmcs/* /var/www/whmcs/
cp /tmp/whmcs/whmcs/cli/clear-cache.php /var/www/whmcs/whmcs/cli/ 2>/dev/null || true
```

### Step 6: Run Database Update
1. Navigate to `https://yourdomain.com/whmcs/install/upgrade.php`
2. Or use command line:
```bash
cd /var/www/whmcs
php -q install/upgrade.php
```

### Step 7: Clear Cache
```bash
cd /var/www/whmcs
rm -rf templates_c/*
rm -rf cache/*
php -q whmcs/cli/clear-cache.php
```

### Step 8: Review Breaking Changes
Check release notes for:
- Deprecated functions
- Database schema changes
- Required PHP version changes
- Configuration file updates

### Step 9: Test Functionality
1. Disable Maintenance Mode
2. Test admin panel features
3. Test client area
4. Test payment processing
5. Test email sending
6. Verify cron jobs

### Step 10: Post-Upgrade Tasks
```bash
# Update file permissions
chown -R www-data:www-data /var/www/whmcs
chmod 644 /var/www/whmcs/configuration.php

# Verify system health
cd /var/www/whmcs
php -q whmcs/cli/audit.php
```

## Rollback Procedure
```bash
# Stop - restore if issues found
rm -rf /var/www/whmcs
mv whmcs_backup_pre_upgrade_* /var/www/whmcs
mysql -u root -p whmcs < whmcs_db_backup_*.sql
```

## Version-Specific Notes
- Always check compatibility matrix
- Some versions require intermediate upgrades
- Check add-on module compatibility

## Verification Checklist
- [ ] Admin area loads correctly
- [ ] All modules functional
- [ ] Payments process correctly
- [ ] Emails send properly
- [ ] No PHP errors in logs
- [ ] System Health shows green

## Tags
- upgrade
- version-update
- maintenance