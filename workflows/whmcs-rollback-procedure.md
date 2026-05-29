# WHMCS Rollback Procedure Workflow

## Overview
This workflow provides step-by-step instructions for rolling back a WHMCS module deployment if issues are detected.

## Prerequisites
- Backup of previous version
- SSH access to server
- Database access

## Step-by-Step Guide

### Step 1: Detect Issues
```bash
# Check for errors in logs
tail -100 /var/www/whmcs/admin/logs/*.log | grep -i error | tail -20

# Check module status
curl -s "https://your-whmcs.com/admin/modules/addons/yourmodule/status.php"

# Check error rates in monitoring
# Review user reports
```

### Step 2: Decision to Rollback
```markdown
# Rollback Decision Checklist

## Issue Severity
- [ ] Critical - Module causing system errors
- [ ] High - Major functionality broken
- [ ] Medium - Important feature not working
- [ ] Low - Minor issue, can wait

## Rollback Criteria
- [ ] Issue affects production users
- [ ] No quick fix available
- [ ] User impact significant
- [ ] Rollback safer than hotfix
```

### Step 3: Create Rollback Script
```bash
#!/bin/bash
# rollback.sh

set -e

MODULE_NAME="yourmodule"
MODULE_DIR="/var/www/whmcs/html/modules/addons/$MODULE_NAME"
BACKUP_DIR="/var/www/whmcs/backups/modules/$MODULE_NAME"

echo "=== WHMCS Module Rollback ==="

# Find most recent backup
LATEST_BACKUP=$(ls -td "$BACKUP_DIR"/backup_* 2>/dev/null | head -1)

if [ -z "$LATEST_BACKUP" ]; then
    echo "ERROR: No backup found"
    exit 1
fi

echo "Rolling back to: $LATEST_BACKUP"

# Deactivate module first
echo "Deactivating module..."
php "$MODULE_DIR/scripts/deactivate.php" 2>/dev/null || true

# Remove current version
echo "Removing current version..."
rm -rf "$MODULE_DIR"

# Restore from backup
echo "Restoring from backup..."
cp -r "$LATEST_BACKUP" "$MODULE_DIR"

# Set permissions
echo "Setting permissions..."
chmod -R 755 "$MODULE_DIR"

# Clear cache
echo "Clearing cache..."
rm -rf /var/www/whmcs/admin/downloads/cache/* 2>/dev/null || true

# Verify
echo "Verifying rollback..."
if [ -f "$MODULE_DIR/$MODULE_NAME.php" ]; then
    echo "[OK] Module file restored"
else
    echo "[FAIL] Module file missing"
    exit 1
fi

echo "=== Rollback Complete ==="
echo "Please re-activate module in WHMCS Admin"
```

### Step 4: Execute Rollback
```bash
# Make script executable
chmod +x rollback.sh

# Execute rollback
./rollback.sh
```

### Step 5: Verify Rollback
```bash
# Check module file
ls -la /var/www/whmcs/html/modules/addons/yourmodule/

# Check version
grep "VERSION" /var/www/whmcs/html/modules/addons/yourmodule/yourmodule.php

# Reactivate module via WHMCS Admin
# Navigate to Configuration > Module Settings
# Deactivate and reactivate

# Test basic functionality
curl -s "https://your-whmcs.com/admin/modules/addons/yourmodule/test.php"
```

### Step 6: Post-Rollback
```markdown
# Post-Rollback Actions

## Immediate
- [ ] Module reactivated
- [ ] Basic functionality verified
- [ ] Error rates normal
- [ ] Users notified

## Investigation
- [ ] Root cause identified
- [ ] Issue documented
- [ ] Fix planned

## Recovery
- [ ] Fix developed and tested
- [ ] Staging deployment planned
- [ ] Communication prepared
```

## Rollback Checklist

### Initiation
- [ ] Issue confirmed
- [ ] Rollback approved
- [ ] Team notified

### Execution
- [ ] Backup located
- [ ] Module deactivated
- [ ] Files restored
- [ ] Permissions set

### Verification
- [ ] Module file exists
- [ ] Version correct
- [ ] Module activates
- [ ] Features work

### Communication
- [ ] Team updated
- [ ] Users notified
- [ ] Status page updated
