# WHMCS Error Investigation Workflow

## Overview
This workflow guides you through investigating errors in WHMCS modules.

## Prerequisites
- Error logs access
- WHMCS admin access
- Debug mode enabled

## Step-by-Step Guide

### Step 1: Collect Error Information
```markdown
# Error Report Template

## Error Details
- Error Type:
- Error Message:
- Timestamp:
- User Affected:
- Request ID:

## Steps to Reproduce
1.
2.
3.

## Environment
- WHMCS Version:
- Module Version:
- PHP Version:
- Browser:

## Error Logs
```
[paste error logs here]
```

## Additional Context
[paste any additional information]
```

### Step 2: Check Error Logs
```bash
# WHMCS logs
tail -100 /var/www/whmcs/admin/logs/*.log

# Module logs
tail -100 /var/www/whmcs/storage/logs/module.log

# PHP error log
tail -100 /var/log/php-fpm/www-error.log

# Apache/Nginx error log
tail -100 /var/log/apache2/error.log
```

### Step 3: Enable Debug Mode
```php
// In configuration.php
define('WHMCS_DEBUG', true);
ini_set('display_errors', 1);
error_reporting(E_ALL);
```

### Step 4: Reproduce Error
```bash
# From command line
php /var/www/whmcs/modules/addons/yourmodule/test.php

# Via curl
curl -v "https://whmcs.com/admin/modules/addons/yourmodule/test.php"
```

### Step 5: Analyze Stack Trace
```php
// Add to error handler
set_exception_handler(function($e) {
    error_log("Exception: " . $e->getMessage());
    error_log("Stack: " . $e->getTraceAsString());
    // Show detailed error in debug mode
    if (defined('WHMCS_DEBUG') && WHMCS_DEBUG) {
        echo "<pre>" . $e . "</pre>";
    }
});
```

## Error Investigation Checklist

### Collection
- [ ] Error message captured
- [ ] Stack trace obtained
- [ ] Environment documented
- [ ] Steps to reproduce recorded

### Analysis
- [ ] Root cause identified
- [ ] Related code located
- [ ] Impact assessed
- [ ] Similar issues checked

### Resolution
- [ ] Fix developed
- [ ] Fix tested
- [ ] Deployed
- [ ] Monitored
