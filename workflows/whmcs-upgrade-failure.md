# WHMCS Upgrade Failure Debug Workflow

## Overview
This workflow guides you through debugging WHMCS module upgrade failures.

## Prerequisites
- Previous version backup
- Upgrade logs

## Step-by-Step Guide

### Step 1: Check Upgrade Logs
```bash
# View module upgrade log
tail -100 /var/www/whmcs/storage/logs/module_upgrade.log

# View WHMCS activity log
tail -100 /var/www/whmcs/admin/logs/activity.log | grep -i upgrade
```

### Step 2: Verify Current Version
```php
// Check current module version
$version = \WHMCS\Database\Capsule::table('tbladdonmodules')
    ->where('module', 'yourmodule')
    ->value('version');

echo "Current version: $version";
```

### Step 3: Restore Previous Version
```bash
#!/bin/bash
# restore-previous.sh

MODULE_DIR="/var/www/whmcs/modules/addons/yourmodule"
BACKUP_DIR="/var/www/whmcs/backups/modules/yourmodule"

# Find most recent backup
BACKUP=$(ls -td "$BACKUP_DIR"/backup_* | head -1)

if [ -z "$BACKUP" ]; then
    echo "No backup found"
    exit 1
fi

echo "Restoring from: $BACKUP"

# Restore files
rm -rf "$MODULE_DIR"
cp -r "$BACKUP" "$MODULE_DIR"

# Update version in database
mysql -u whmcs_user -p whmcs_db -e "
    UPDATE tbladdonmodules 
    SET version = 'previous_version' 
    WHERE module = 'yourmodule'
"

echo "Restoration complete"
```

### Step 4: Debug Upgrade Script
```php
// In upgrade script
function upgrade_module_2_0_0($vars)
{
    $currentVersion = $vars['version'];
    
    logActivity("Starting upgrade from $currentVersion to 2.0.0");
    
    try {
        // Run migrations
        Migration::run();
        
        // Update configuration
        update_config();
        
        // Clear cache
        clear_cache();
        
        logActivity("Upgrade to 2.0.0 completed");
        
    } catch (Exception $e) {
        logActivity("Upgrade failed: " . $e->getMessage());
        throw $e;
    }
}
```

## Upgrade Failure Debug Checklist

### Investigation
- [ ] Upgrade logs reviewed
- [ ] Errors identified
- [ ] Current state verified
- [ ] Backup available

### Resolution
- [ ] Previous version restored
- [ ] Issue identified
- [ ] Fix developed
- [ ] Upgrade re-attempted
