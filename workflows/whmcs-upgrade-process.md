# WHMCS Version Upgrade Workflow

## Purpose
Upgrade WHMCS to the latest version safely

## Prerequisites
- WHMCS installed
- SSH/cPanel access
- Current WHMCS version
- Full backup

## Step 1: Check Current Version

Navigate to: Utilities > System > Health Status

Note current version: `WHMC X.X.X`

Check for available updates: https://docs.whmcs.com/Changelog

## Step 2: Review Requirements

Verify server meets requirements for new version:
- PHP version
- MySQL version
- Required PHP extensions

## Step 3: Create Complete Backup

```bash
# Database backup
mysqldump -u root -p whmcs_db | gzip > /backup/whmcs_db_$(date +%Y%m%d).sql.gz

# Files backup
tar -czf /backup/whmcs_files_$(date +%Y%m%d).tar.gz /var/www/whmcs --exclude=/var/www/whmcs/templates_c --exclude=/var/www/whmcs/cache

# Verify backups
gunzip -t /backup/whmcs_db_$(date +%Y%m%d).sql.gz
tar -tzf /backup/whmcs_files_$(date +%Y%m%d).tar.gz | head
```

## Step 4: Enable Maintenance Mode

Navigate to: Setup > General Settings > Maintenance Mode

Enable with message: "System upgrade in progress"

## Step 5: Stop Cron Jobs

```bash
# Comment out cron entries
crontab -e

# Add # before WHMCS cron:
# 0,30 * * * * /usr/bin/php /var/www/whmcs/crons/cron.php
```

## Step 6: Download Latest Version

### Method A: Direct Download
```bash
cd /tmp
wget https://downloads.whmcs.com/whmcs-latest.zip
unzip -o whmcs-latest.zip
```

### Method B: Via WHMCS Admin
1. Navigate to: Utilities > System > Check for Updates
2. Click "Download Update"
3. Upload to server

## Step 7: Extract Update Files

```bash
cd /var/www/whmcs

# Extract to temp directory
unzip -o /tmp/whmcs-latest.zip -d /tmp/whmcs_update/

# Copy new files
cp -r /tmp/whmcs_update/whmcs/* .

# Preserve custom files
cp -r /tmp/whmcs_update/whmcs/install/ ./install/
```

## Step 8: Set Permissions

```bash
cd /var/www/whmcs

# Set ownership
chown -R www-data:www-data .

# Set base permissions
find . -type d -exec chmod 755 {} \;
find . -type f -exec chmod 644 {} \;

# Special permissions
chmod 400 configuration.php
chmod 755 templates_c
chmod 755 cache
chmod 755 downloads
chmod 755 attachments
chmod 755 logs
```

## Step 9: Run Database Update

### Automatic (via browser)
Navigate to: `https://yourdomain.com/install/`

WHMCS will detect and run updates automatically.

### Manual (via CLI)
```bash
cd /var/www/whmcs
/usr/bin/php -d register_argc_argv=1 crons/update.php
```

## Step 10: Verify Update

Navigate to: Utilities > System > Health Status

Confirm new version installed.

## Step 11: Clear Cache

Navigate to: Utilities > System > Clear Cache

```bash
rm -rf /var/www/whmcs/templates_c/*
rm -rf /var/www/whmcs/cache/*
```

## Step 12: Test Functionality

Test all critical functions:
1. **Login** - Admin and client
2. **Orders** - Place test order
3. **Payments** - Process test payment
4. **Provisioning** - Verify service creation
5. **Support** - Create test ticket
6. **Cron** - Run manually and verify

## Step 13: Re-enable Cron Jobs

```bash
crontab -e

# Remove # from WHMCS cron entries
0,30 * * * * /usr/bin/php /var/www/whmcs/crons/cron.php
```

## Step 14: Disable Maintenance Mode

Navigate to: Setup > General Settings > Maintenance Mode

Disable maintenance mode.

## Step 15: Monitor System

After upgrade, monitor:
- Error logs
- Performance
- User reports
- Module compatibility

## Step 16: Update Custom Modules

Check and update any custom modules for compatibility with new version.

## Upgrade Checklist

- [ ] Current version checked
- [ ] Requirements verified
- [ ] Full backup created
- [ ] Maintenance mode enabled
- [ ] Cron jobs stopped
- [ ] Latest version downloaded
- [ ] Update files extracted
- [ ] Permissions set
- [ ] Database updated
- [ ] Update verified
- [ ] Cache cleared
- [ ] Functionality tested
- [ ] Cron jobs re-enabled
- [ ] Maintenance mode disabled
- [ ] System monitored
- [ ] Modules updated
