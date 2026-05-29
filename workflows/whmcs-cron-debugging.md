# WHMCS Cron Debug Workflow

## Overview
This workflow guides you through debugging WHMCS cron job issues.

## Prerequisites
- Cron access
- WHMCS admin access

## Step-by-Step Guide

### Step 1: Check Cron Configuration
```bash
# View crontab
crontab -l

# WHMCS cron typically:
# * * * * * php -q /var/www/whmcs/crons/cron.php
```

### Step 2: Run Cron Manually
```bash
# Run full cron
php -q /var/www/whmcs/crons/cron.php

# Run with debug
php -d display_errors=1 /var/www/whmcs/crons/cron.php

# Run specific cron module
php -q /var/www/whmcs/crons/cron.php --action=sync
```

### Step 3: Check Cron Logs
```bash
# View activity log
tail -100 /var/www/whmcs/admin/logs/activity.log | grep -i cron

# Check module cron logs
tail -100 /var/www/whmcs/storage/logs/module_cron.log
```

### Step 4: Test Module Cron
```php
// In your module cron file
<?php
// modules/addons/yourmodule/cron.php

require_once __DIR__ . '/../../../init.php';

use Vendor\Module\CronService;

$service = new CronService();
$service->run();

// Log completion
logActivity("YourModule cron completed");
```

### Step 5: Common Cron Issues
```php
// Issue: Cron not running
// Fix: Verify crontab entry, check permissions

// Issue: Cron running but tasks not executing
// Fix: Check WHMCS cron settings, add logging

// Issue: Duplicate cron execution
// Fix: Implement cron lock file
```

## Cron Debug Checklist

### Investigation
- [ ] Crontab verified
- [ ] Manual execution tested
- [ ] Logs reviewed
- [ ] Task output examined

### Resolution
- [ ] Cron entry fixed
- [ ] Permissions corrected
- [ ] Task logic debugged
- [ ] Scheduling verified
