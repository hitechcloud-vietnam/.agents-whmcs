# WHMCS Bug Fix Workflow

## Purpose

Systematic approach to identifying, fixing, and verifying bugs in WHMCS installations and custom modules. Ensures reproducible fixes with proper documentation and regression prevention.

## Prerequisites

- WHMCS installation access (admin panel and file system)
- Development environment mirroring production
- Access to WHMCS error logs and system logs
- Version control system for tracking changes

## Workflow Steps

### Step 1: Bug Report Analysis

Collect all available information about the bug:

```php
// Check WHMCS activity log for related entries
// Location: /var/log/whmcs/activity.log or WHMCS Admin > System > Activity Log

// Enable debug mode in configuration.php for detailed error output
// Add to end of configuration.php:
$debug = true;
```

Review the bug report for:
- Steps to reproduce
- Expected behavior vs actual behavior
- Affected WHMCS version
- Browser/environment details
- Error messages or screenshots

### Step 2: Environment Setup

Create a safe testing environment:

```bash
# Clone production database for local testing
mysqldump -u root -p production_whmcs > bug_test_$(date +%Y%m%d).sql

# Set up local WHMCS copy
cp -r /var/www/production-whmcs /var/www/local-whmcs
mysql -u root -p local_whmcs < bug_test_$(date +%Y%m%d).sql

# Update configuration.php for local environment
# Change $mysql_host, $mysql_username, $mysql_password, $mysql_database
# Update $domain, $systems_url
```

### Step 3: Reproduce the Bug

Attempt to reproduce the issue in the controlled environment:

```php
// Add temporary debug logging in suspected code areas
// File: /includes/debug.php (create if not exists)

function debug_log($message, $data = []) {
    $logFile = __DIR__ . '/../storage/logs/bug_debug.log';
    $timestamp = date('Y-m-d H:i:s');
    $entry = "[$timestamp] $message: " . print_r($data, true) . "\n";
    file_put_contents($logFile, $entry, FILE_APPEND);
}

// Use in code:
debug_log('Variable state', ['var1' => $var1, 'var2' => $var2]);
```

Enable WHMCS debug logging:

```php
// In configuration.php
$debug = true;
$display_errors = true;

// In includes/config.php, temporarily set:
defined('WHLMCS_DEBUG_LOGGING') || define('WHLMCS_DEBUG_LOGGING', true);
```

### Step 4: Identify Root Cause

Systematically narrow down the source:

```php
// Use WHMCS built-in logging
logActivity('Debug: Processing ticket #' . $ticketId);

// Check hook execution order
// File: /includes/hooks/ - review custom hooks

// Use Xdebug for step-through debugging
// Configuration for VS Code:
{
    "name": "Xdebug",
    "type": "php",
    "request": "launch",
    "port": 9000,
    "pathMappings": {
        "/var/www/whmcs": "${workspaceFolder}"
    }
}
```

Common bug sources in WHMCS:
- Hook execution order conflicts
- Database query failures or deadlocks
- Cache inconsistencies
- Third-party module interference
- PHP version compatibility issues
- Memory limit exceeded

### Step 5: Implement Fix

Apply the correction with version control:

```bash
# Create feature branch for the fix
git checkout -b bugfix/issue-description

# Make the fix
# Example: Fix for missing invoice line item
```

```php
// Before fix (example)
foreach ($items as $item) {
    $invoice->addItem($item['name'], $item['amount']);
}

// After fix - add validation
foreach ($items as $item) {
    if (empty($item['name']) || !is_numeric($item['amount'])) {
        logActivity('Invalid invoice item skipped: ' . json_encode($item));
        continue;
    }
    $invoice->addItem($item['name'], $item['amount']);
}
```

### Step 6: Test the Fix

Verify the fix works and doesn't break existing functionality:

```php
// Create unit test for the fix
// File: /resources/testing/Unit/BugFixTest.php

namespace WHMCS\Testing\Unit;

class BugFixTest extends \PHPUnit\Framework\TestCase
{
    public function testInvoiceItemValidation()
    {
        $invoice = new \WHMCS\Invoice();
        $result = $invoice->addItem('', 100.00);
        $this->assertFalse($result);
        
        $result = $invoice->addItem('Valid Item', 'not-a-number');
        $this->assertFalse($result);
        
        $result = $invoice->addItem('Valid Item', 100.00);
        $this->assertTrue($result);
    }
}
```

Run tests:

```bash
cd /var/www/whmcs
php vendor/bin/phpunit resources/testing/Unit/BugFixTest.php
```

### Step 7: Deploy Fix

Apply the fix to production following change management:

```bash
# Test on staging first
rsync -avz --exclude='configuration.php' \
    /var/www/local-whmcs/ \
    user@staging-server:/var/www/staging-whmcs/

# After staging verification, deploy to production
# Use your deployment script or process
```

```php
// Always backup before making changes
// Create complete backup including database
$backupDir = '/var/backups/whmcs/bugfix-' . date('Y-m-d-His');
```

### Step 8: Verify in Production

Confirm the fix works in the live environment:

```php
// Add temporary production verification
// In the fixed file, add:
if ($whmcs->get_config('debug')) {
    logActivity('Bug fix verification: ' . __FUNCTION__ . ' executed successfully');
}

// After verification, remove debug code
```

Check:
- The specific bug is resolved
- No new errors in WHMCS logs
- Related functionality still works
- Performance hasn't degraded

## Verification Checklist

- [ ] Bug is reproducible in development environment
- [ ] Root cause identified and documented
- [ ] Fix implemented with version control
- [ ] Unit tests created and passing
- [ ] Integration tests verify fix doesn't break other features
- [ ] Staging environment tested successfully
- [ ] Production deployment completed with backup
- [ ] Bug confirmed fixed in production
- [ ] Debug code removed from production
- [ ] Documentation updated if needed
- [ ] Monitoring in place for recurrence detection

## Related Skills and Documentation

- [WHMCS Module Development](whmcs-module-development-workflow.md)
- [WHMCS Security Audit](whmcs-security-audit.md)
- [WHMCS Module Testing](whmcs-module-testing.md)
- [WHMCS Performance Audit](whmcs-performance-audit.md)
- WHMCS Documentation: https://developers.whmcs.com/
- WHMCS Hook System: https://developers.whmcs.com/advanced/hooks/

## Notes

- Always create a backup before applying any fix to production
- Document any temporary files or code added during debugging for cleanup
- Consider creating automated tests to prevent regression
- If the bug affects multiple WHMCS versions, plan fixes for each
- Report significant bugs to WHMCS via their support portal if applicable
