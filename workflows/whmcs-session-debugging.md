# WHMCS Session Debug Workflow

## Overview
This workflow guides you through debugging session-related issues.

## Prerequisites
- PHP session configuration
- Browser developer tools

## Step-by-Step Guide

### Step 1: Check Session Configuration
```php
// In configuration.php or module
echo "Session save path: " . ini_get('session.save_path');
echo "Session name: " . ini_get('session.name');
echo "Session lifetime: " . ini_get('session.gc_maxlifetime');
```

### Step 2: Enable Session Logging
```php
public function setSessionData(string $key, $value): void
{
    if (!isset($_SESSION[$key])) {
        logModuleCall('yourmodule', 'session_set', "$key = " . print_r($value, true), '');
    }
    $_SESSION[$key] = $value;
}

public function getSessionData(string $key)
{
    $value = $_SESSION[$key] ?? null;
    logModuleCall('yourmodule', 'session_get', "key: $key", "value: " . print_r($value, true));
    return $value;
}
```

### Step 3: Check Session in Browser
```javascript
// In browser console
console.log('Session:', <?= json_encode($_SESSION) ?>);

// Check cookies
document.cookie.split(';').forEach(cookie => {
    console.log(cookie.trim());
});
```

### Step 4: Common Session Issues
```php
// Issue: Session not starting
// Fix: Check session.auto_start configuration

// Issue: Session expiring too quickly
// Fix: Adjust session.gc_maxlifetime

// Issue: Session data lost
// Fix: Check session.save_path permissions

// Issue: Session hijacking
// Fix: Regenerate session ID on login
```

## Session Debug Checklist

### Investigation
- [ ] Session configuration verified
- [ ] Cookies checked in browser
- [ ] Session data logged
- [ ] Expiration tested

### Resolution
- [ ] Session path writable
- [ ] Lifetime appropriate
- [ ] Data persisting correctly
- [ ] Regeneration working
