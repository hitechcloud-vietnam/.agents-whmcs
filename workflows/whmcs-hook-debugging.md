# WHMCS Hook Debug Workflow

## Overview
This workflow guides you through debugging WHMCS hook execution issues.

## Prerequisites
- Hook implementation
- WHMCS admin access

## Step-by-Step Guide

### Step 1: List Registered Hooks
```php
// In WHMCS Admin > Utilities > Logs > Module Debug
// Or via API
$hooks = \App::getRegisteredHooks();
print_r($hooks);
```

### Step 2: Add Hook Debugging
```php
// Add to your hook function
function yourmodule_client_add($vars)
{
    $clientId = $vars['userid'] ?? null;
    
    logModuleCall(
        'yourmodule',
        'hook_client_add',
        "Client add hook triggered",
        "userid: $clientId"
    );
    
    // Your logic
    if (!$clientId) {
        logModuleCall(
            'yourmodule',
            'hook_client_add',
            '',
            'ERROR: No userid provided'
        );
        return;
    }
    
    // Process...
}
```

### Step 3: Test Hook Manually
```bash
#!/bin/bash
# test-hook.sh

# Simulate ClientAdd hook
php -r "
\$_POST = ['userid' => 1];
\$_GET = [];
require '/var/www/whmcs/init.php';

\$vars = ['userid' => 1, 'firstname' => 'Test'];
run_hooks('ClientAdd', \$vars);

echo 'Hook executed';
"
```

### Step 4: Common Hook Issues
```php
// Issue: Hook not executing
// Fix: Check hook is registered, module activated

// Issue: Hook causing errors
// Fix: Wrap in try-catch, log errors

// Issue: Hook modifying wrong data
// Fix: Return correct format, check variable names
```

## Hook Debug Checklist

### Investigation
- [ ] Hook registered
- [ ] Module activated
- [ ] Logs reviewed
- [ ] Execution tested

### Resolution
- [ ] Hook registered correctly
- [ ] Logic fixed
- [ ] Errors caught
- [ ] Data correct
