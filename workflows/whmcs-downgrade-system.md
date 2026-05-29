# WHMCS System Downgrade Workflow

## Description
Step-by-step guide for downgrading WHMCS to a previous version.

## Warning
Downgrading WHMCS is not officially supported. Only attempt if absolutely necessary.

## Prerequisites
- Full backup of current installation
- Target version WHMCS files
- Database compatible with target version
- Maintenance mode ready

## Steps

### Step 1: Create Complete Backup
```bash
# Backup entire installation
cd /var/www
tar -czvf whmcs_pre_downgrade_$(date +%Y%m%d).tar.gz whmcs/

# Backup database
mysqldump -u root -p whmcs > whmcs_db_pre_downgrade_$(date +%Y%m%d).sql
```

### Step 2: Export Critical Data
```bash
# Export to CSV via WHMCS admin or direct SQL
mysqldump -u root -p whmcs \
  --tables tblclients tblinvoices tblhosting \
  tblproducts tbllogins tblticket \
  > critical_data_backup.sql
```

### Step 3: Enable Maintenance Mode
1. Login to WHMCS Admin
2. Enable Maintenance Mode
3. Backup any custom files

### Step 4: Prepare Target Version
```bash
cd /tmp
# Download target version (example: 8.2.0)
wget https://download.whmcs.com/whmcs.zip
unzip whmcs.zip
```

### Step 5: Backup and Replace Files
```bash
# Move current to backup
mv /var/www/whmcs /var/www/whmcs_v8_current

# Extract target version
unzip whmcs.zip -d /var/www/
mv /var/www/whmcs /var/www/whmcs_v8_target
```

### Step 6: Restore Configuration
```bash
# Copy configuration from backup
cp /var/www/whmcs_v8_current/configuration.php /var/www/whmcs_v8_target/
cp -r /var/www/whmcs_v8_current/{attachments,downloads,language,files,templates} /var/www/whmcs_v8_target/
```

### Step 7: Restore Customizations
```bash
# Restore custom modules
cp -r /var/www/whmcs_v8_current/modules/* /var/www/whmcs_v8_target/modules/

# Restore custom templates
cp -r /var/www/whmcs_v8_current/templates/* /var/www/whmcs_v8_target/templates/
```

### Step 8: Restore Database
```bash
# Drop existing database
mysql -u root -p -e "DROP DATABASE IF EXISTS whmcs;"
mysql -u root -p -e "CREATE DATABASE whmcs CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# Restore database (note: may need manual adjustments for schema differences)
mysql -u root -p whmcs < whmcs_db_pre_downgrade_*.sql
```

### Step 9: Fix Permissions
```bash
cd /var/www/whmcs_v8_target
chown -R www-data:www-data .
chmod 755 .
chmod 644 configuration.php
chmod 755 templates_c attachments downloads language
```

### Step 10: Run Database Compatibility Script
```bash
# May need manual SQL adjustments for schema changes
# Check WHMCS logs for errors
tail -100 /var/www/whmcs_v8_target/logs/*.log
```

### Step 11: Test System
1. Disable Maintenance Mode
2. Test admin login
3. Test basic functionality
4. Verify data integrity

## Manual Schema Adjustments
```sql
-- Check for missing columns after downgrade
-- May need to add columns back or remove incompatible data
```

## Common Issues
- Missing database columns
- Template incompatibilities
- Module version mismatches
- Configuration format changes

## Rollback
```bash
# Restore to current version
rm -rf /var/www/whmcs_v8_target
mv /var/www/whmcs_v8_current /var/www/whmcs
mysql -u root -p whmcs < whmcs_db_pre_downgrade_*.sql
```

## Recommendations
- Consider reinstalling with clean database
- Import data using WHMCS import tools
- Contact WHMCS support for guidance

## Tags
- downgrade
- version-rollback
- recovery