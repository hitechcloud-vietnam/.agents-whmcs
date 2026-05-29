# WHMCS Patch Application Workflow

## Purpose
Apply security patches and hotfixes to WHMCS

## Prerequisites
- WHMCS installed
- SSH access
- Backup of current installation

## Step 1: Identify Patch Required

### Check WHMCS Security Page
Visit: https://whmcs.com/security/

### Check WHMCS Admin Notifications
Navigate to: Utilities > System Notifications

### Check Version
Navigate to: Utilities > System Health Status

## Step 2: Read Patch Documentation

Before applying any patch:
1. Read release notes
2. Understand what the patch fixes
3. Check if affected by the vulnerability
4. Note any prerequisites

## Step 3: Create Full Backup

```bash
# Database backup
mysqldump -u root -p whmcs_db | gzip > /backup/whmcs_patch_$(date +%Y%m%d).sql.gz

# Files backup
tar -czf /backup/whmcs_patch_$(date +%Y%m%d).tar.gz /var/www/whmcs --exclude=/var/www/whmcs/templates_c --exclude=/var/www/whmcs/cache

# Verify backup
tar -tzf /backup/whmcs_patch_$(date +%Y%m%d).tar.gz | head
gunzip -t /backup/whmcs_patch_$(date +%Y%m%d).sql.gz
```

## Step 4: Enable Maintenance Mode

Navigate to: Setup > General Settings > Maintenance Mode

Enable with message: "Security patch being applied"

## Step 5: Stop Cron Jobs

```bash
crontab -e

# Comment out WHMCS cron
# 0,30 * * * * /usr/bin/php /var/www/whmcs/crons/cron.php
```

## Step 6: Download Patch

### From WHMCS
```bash
cd /tmp
wget https://downloads.whmcs.com/patches/[patch-file].zip
```

### From Admin Download
1. Navigate to: Utilities > System > Check for Updates
2. Download security patch
3. Upload to server

## Step 7: Verify Patch Integrity

```bash
# Check patch file
ls -la /tmp/[patch-file].zip

# Verify checksum (if provided)
sha256sum /tmp/[patch-file].zip
```

## Step 8: Apply Patch

### Standard Patch
```bash
cd /var/www/whmcs
unzip -o /tmp/[patch-file].zip
```

### Manual File Replacement
If patch contains specific files:
```bash
# Backup existing files
cp -r /var/www/whmcs/[affected-file] /var/www/whmcs/[affected-file].bak

# Replace with patched version
cp /tmp/patch/[affected-file] /var/www/whmcs/[affected-file]

# Restore permissions
chown www-data:www-data /var/www/whmcs/[affected-file]
chmod 644 /var/www/whmcs/[affected-file]
```

## Step 9: Database Patch (if included)

```bash
# Run SQL patch
mysql -u root -p whmcs_db < /tmp/patch/database_patch.sql

# Or via PHP
/usr/bin/php -r "include '/var/www/whmcs/db/patch.php';"
```

## Step 10: Set Permissions

```bash
cd /var/www/whmcs

# Reset all permissions
chown -R www-data:www-data .
find . -type d -exec chmod 755 {} \;
find . -type f -exec chmod 644 {} \;

# Critical files
chmod 400 configuration.php
chmod 755 admin
chmod 755 templates_c
chmod 755 cache
```

## Step 11: Clear Cache

```bash
rm -rf /var/www/whmcs/templates_c/*
rm -rf /var/www/whmcs/cache/*
```

## Step 12: Verify Patch Applied

```bash
# Check patched file
diff /var/www/whmcs/[affected-file].bak /var/www/whmcs/[affected-file]

# Or check version
grep -i "version" /var/www/whmcs/includes/version.php
```

## Step 13: Test Functionality

1. **Admin Login**
   - Test admin area access
   - Verify all admin functions work

2. **Client Login**
   - Test client area
   - Test ordering

3. **Run Cron**
   ```bash
   /usr/bin/php /var/www/whmcs/crons/cron.php
   ```

## Step 14: Re-enable Cron

```bash
crontab -e

# Uncomment WHMCS cron
0,30 * * * * /usr/bin/php /var/www/whmcs/crons/cron.php
```

## Step 15: Disable Maintenance Mode

Navigate to: Setup > General Settings > Maintenance Mode

Disable maintenance mode.

## Step 16: Monitor System

After patch:
- Watch error logs
- Monitor performance
- Check user reports

## Step 17: Document Patch

Record in patch log:
```
Date: [date]
Patch Version: [version]
Files Modified: [list]
Database Changes: [list]
Tested By: [name]
Status: [success/failed]
Notes: [any issues]
```

## Patch Application Checklist

- [ ] Patch identified
- [ ] Documentation read
- [ ] Full backup created
- [ ] Maintenance mode enabled
- [ ] Cron stopped
- [ ] Patch downloaded
- [ ] Patch verified
- [ ] Patch applied
- [ ] Database patched (if needed)
- [ ] Permissions set
- [ ] Cache cleared
- [ ] Patch verified
- [ ] Functionality tested
- [ ] Cron re-enabled
- [ ] Maintenance mode disabled
- [ ] System monitored
- [ ] Patch documented
