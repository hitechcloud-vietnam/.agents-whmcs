# WHMCS Error Codes Reference

**Version:** 8.x
**Updated:** 2026-05-28
**Related Skills:** `troubleshooting-guide`, `api-endpoints-reference`

## Overview

This reference documents WHMCS error codes, their meanings, and resolution strategies. Error codes are categorized by component and severity level.

## Error Code Ranges

| Range | Category |
|-------|----------|
| 1000-1999 | Authentication Errors |
| 2000-2999 | Database Errors |
| 3000-3999 | Module Errors |
| 4000-4999 | Payment Errors |
| 5000-5999 | Domain Errors |
| 6000-6999 | API Errors |
| 7000-7999 | Configuration Errors |
| 8000-8999 | System Errors |
| 9000-9999 | Hook Errors |

## Authentication Errors (1000-1999)

### 1001 - Invalid Credentials
**Meaning:** Username or password is incorrect.
**Context:** Admin login, client login, API authentication.
**Resolution:**
```php
// Check if credentials are correct
// Verify password hash matches
// Check if account is locked or disabled

// Admin login failure tracking
if ($failedAttempts > 5) {
    // Account may be locked
}
```

### 1002 - Account Locked
**Meaning:** Account has been locked due to too many failed attempts.
**Context:** Brute force protection triggered.
**Resolution:**
```php
// Wait for lockout period (usually 15-30 minutes)
// Or contact admin to unlock

// Check tbladmin_logins for lockout status
$locked = Capsule::table('tbladmin_logins')
    ->where('username', $username)
    ->where('success', 0)
    ->where('date', '>', date('Y-m-d H:i:s', strtotime('-15 minutes')))
    ->count();
```

### 1003 - 2FA Verification Failed
**Meaning:** Two-factor authentication code is invalid or expired.
**Resolution:**
```php
// Verify 2FA code is correct
// Check device time synchronization
// Use backup codes if available
```

### 1004 - Session Expired
**Meaning:** User session has timed out.
**Resolution:**
```php
// Re-authenticate
// Increase session timeout in configuration
// Check for session storage issues
```

### 1005 - IP Not Allowed
**Meaning:** Access denied due to IP restrictions.
**Resolution:**
```php
// Check IP whitelist settings
// Verify IP is not in blacklist
// Contact admin for IP approval
```

## Database Errors (2000-2999)

### 2001 - Connection Failed
**Meaning:** Unable to connect to database server.
**Resolution:**
```bash
# Check MySQL service
systemctl status mysql

# Verify credentials in configuration.php
# Check firewall rules
# Verify MySQL is listening
```

### 2002 - Database Not Found
**Meaning:** Specified database does not exist.
**Resolution:**
```bash
# Create database
mysql -u root -p -e "CREATE DATABASE whmcs CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# Verify database name in configuration.php
```

### 2003 - Query Failed
**Meaning:** SQL query execution failed.
**Resolution:**
```sql
-- Check query syntax
-- Verify table exists
-- Check for missing indexes

-- Enable query logging
SET GLOBAL general_log = 'ON';
```

### 2004 - Duplicate Entry
**Meaning:** Attempted to insert duplicate value in unique column.
**Resolution:**
```php
// Check for existing records before insert
$exists = Capsule::table('tblclients')
    ->where('email', $email)
    ->exists();

if ($exists) {
    throw new Exception('Client with this email already exists');
}
```

### 2005 - Table Locked
**Meaning:** Table is locked by another query.
**Resolution:**
```sql
-- Check for long-running queries
SHOW FULL PROCESSLIST;

-- Wait and retry
-- Optimize slow queries
```

### 2006 - Foreign Key Constraint
**Meaning:** Referenced record does not exist.
**Resolution:**
```sql
-- Verify parent record exists
-- Check foreign key relationships
-- Disable foreign key checks temporarily (not recommended)
```

## Module Errors (3000-3999)

### 3001 - Module Not Found
**Meaning:** Requested module does not exist or is not installed.
**Resolution:**
```bash
# Check module directory
ls -la /path/to/whmcs/modules/servers/

# Verify module files are complete
# Reinstall module if necessary
```

### 3002 - Module Function Missing
**Meaning:** Required module function is not implemented.
**Resolution:**
```php
// Check if required function exists in module file
// Functions required for server modules:
// - CreateAccount
// - SuspendAccount
// - UnsuspendAccount
// - TerminateAccount

// Verify function name spelling
```

### 3003 - Module API Error
**Meaning:** Third-party API returned an error.
**Resolution:**
```php
// Check API credentials
// Verify API endpoint URL
// Check API rate limits
// Review API documentation

// Log API response for debugging
logModuleCall('modulename', 'function', $params, $response, $error);
```

### 3004 - Module Configuration Error
**Meaning:** Module configuration is incomplete or invalid.
**Resolution:**
```php
// Verify all required config options are set
// Check server credentials
// Review module settings in WHMCS admin
```

### 3005 - Module Timeout
**Meaning:** Module operation timed out.
**Resolution:**
```php
// Increase timeout in module
// Check network connectivity
// Optimize API response handling
// Implement retry logic
```

## Payment Errors (4000-4999)

### 4001 - Invalid Payment Method
**Meaning:** Specified payment gateway is not available.
**Resolution:**
```php
// Verify gateway is installed and enabled
// Check gateway configuration
// Review gateway settings in admin
```

### 4002 - Payment Declined
**Meaning:** Payment was declined by processor.
**Resolution:**
```php
// Check card details
// Verify sufficient funds
// Try alternative payment method
// Contact payment processor
```

### 4003 - Transaction Not Found
**Meaning:** Payment transaction ID not found.
**Resolution:**
```sql
-- Verify transaction in gateway logs
-- Check tblepaytransactions table
-- Review payment gateway callback
```

### 4004 - Amount Mismatch
**Meaning:** Payment amount does not match invoice.
**Resolution:**
```php
// Check invoice total
// Verify payment amount
// Handle partial payments if enabled
// Review currency settings
```

### 4005 - Refund Failed
**Meaning:** Unable to process refund.
**Resolution:**
```php
// Verify refund is allowed
// Check transaction status
// Verify gateway supports refunds
// Check refund amount limits
```

### 4006 - Duplicate Transaction
**Meaning:** Transaction ID already processed.
**Resolution:**
```php
// Check tblepaytransactions for duplicates
// Verify idempotency in payment processing
// Log duplicate attempts
```

## Domain Errors (5000-5999)

### 5001 - Domain Not Available
**Meaning:** Domain is not available for registration.
**Resolution:**
```php
// Check domain availability via WHOIS
// Verify TLD is supported
// Check for trademark issues
```

### 5002 - Transfer Failed
**Meaning:** Domain transfer was rejected or failed.
**Resolution:**
```bash
# Check transfer authorization code
# Verify domain is eligible for transfer
# Check transfer lock status
# Confirm with current registrar
```

### 5003 - Registry Error
**Meaning:** Domain registry returned an error.
**Resolution:**
```php
// Check registry status
// Verify registrar API is working
// Review registry error codes
// Contact registrar support
```

### 5004 - Domain Locked
**Meaning:** Domain is locked by registrar.
**Resolution:**
```php
// Unlock domain at current registrar
// Wait for DNS propagation
// Verify transfer lock status
```

### 5005 - Contact Verification Failed
**Meaning:** Domain contact information failed verification.
**Resolution:**
```php
// Verify contact details are accurate
// Check email is accessible
// Complete domain contact verification
```

## API Errors (6000-6999)

### 6001 - Invalid API Key
**Meaning:** API key is invalid or inactive.
**Resolution:**
```php
// Generate new API key in admin
// Verify key is active
// Check API key permissions
```

### 6002 - API Rate Limited
**Meaning:** Too many API requests.
**Resolution:**
```php
// Implement rate limiting
// Wait before retrying
// Request rate limit increase
```

### 6003 - API Permission Denied
**Meaning:** API key lacks required permissions.
**Resolution:**
```php
// Check API key permissions
// Update API access controls
// Use appropriate API key
```

### 6004 - Invalid API Action
**Meaning:** Requested API action does not exist.
**Resolution:**
```php
// Verify action name
// Check API documentation
// Use correct action parameter
```

### 6005 - Missing Required Parameter
**Meaning:** Required API parameter is missing.
**Resolution:**
```php
// Review required parameters
// Provide all required fields
// Check parameter names
```

### 6006 - Invalid Parameter Value
**Meaning:** API parameter value is invalid.
**Resolution:**
```php
// Check parameter format
// Verify value is within allowed range
// Review parameter documentation
```

## Configuration Errors (7000-7999)

### 7001 - Configuration File Missing
**Meaning:** configuration.php does not exist.
**Resolution:**
```bash
# Restore from backup
# Run WHMCS installer
# Verify file permissions
```

### 7002 - Invalid Configuration
**Meaning:** Configuration values are invalid.
**Resolution:**
```php
// Verify all required config values
// Check configuration.php syntax
// Restore default configuration
```

### 7003 - Encryption Key Invalid
**Meaning:** Database encryption key is invalid.
**Resolution:**
```php
// Restore from backup
// Update $cc_encryption_hash in configuration.php
// Re-encrypt sensitive data if needed
```

### 7004 - Template Not Found
**Meaning:** Requested template does not exist.
**Resolution:**
```bash
# Check template directory
ls -la /path/to/whmcs/templates/

# Verify template is properly installed
# Check template file permissions
```

### 7005 - Language File Missing
**Meaning:** Required language file not found.
**Resolution:**
```bash
# Check language directory
ls -la /path/to/whmcs/lang/

# Upload missing language files
# Verify language is installed
```

## System Errors (8000-8999)

### 8001 - Out of Memory
**Meaning:** PHP memory limit exceeded.
**Resolution:**
```php
// Increase memory_limit in php.ini
ini_set('memory_limit', '512M');

// Optimize code to use less memory
// Clear cache files
// Check for memory leaks
```

### 8002 - Maximum Execution Time
**Meaning:** Script exceeded maximum execution time.
**Resolution:**
```php
// Increase max_execution_time
ini_set('max_execution_time', 300);

// Optimize slow operations
// Implement chunked processing
```

### 8003 - File Upload Failed
**Meaning:** File upload was unsuccessful.
**Resolution:**
```php
// Check upload_max_filesize
// Verify file type is allowed
// Check upload directory permissions
// Verify form has enctype="multipart/form-data"
```

### 8004 - File Not Writable
**Meaning:** Cannot write to file or directory.
**Resolution:**
```bash
# Set correct permissions
chmod 755 /path/to/whmcs/storage
chmod 644 /path/to/whmcs/storage/logs/*.log

# Check ownership
chown -R user:www-data /path/to/whmcs
```

### 8005 - Class Not Found
**Meaning:** Required PHP class does not exist.
**Resolution:**
```php
// Run composer install
// Check autoloader
// Verify file is included
// Clear and regenerate autoload
```

## Hook Errors (9000-9999)

### 9001 - Hook File Syntax Error
**Meaning:** Hook file contains PHP syntax error.
**Resolution:**
```bash
# Check PHP syntax
php -l /path/to/hooks/your_hook.php

# Fix syntax errors
# Disable problematic hook temporarily
```

### 9002 - Hook Function Error
**Meaning:** Hook function threw an exception.
**Resolution:**
```php
// Add try-catch in hook
// Check hook documentation
// Review error logs
// Disable hook temporarily
```

### 9003 - Hook Not Found
**Meaning:** Referenced hook point does not exist.
**Resolution:**
```php
// Verify hook name
// Check WHMCS version supports hook
// Review hook documentation
```

## Error Handling Best Practices

```php
try {
    // Your code
    $result = someOperation();
} catch (\Exception $e) {
    // Log error
    logActivity('Operation failed: ' . $e->getMessage());

    // Return user-friendly error
    return [
        'error' => true,
        'message' => 'An error occurred. Please try again.',
        'code' => $e->getCode(),
    ];
}

// Use WHMCS exception classes
use WHMCS\Exception;

throw new WHMCS\Exception\Error('Custom error message');
throw new WHMCS\Exception\Module\Error('Module operation failed');
```

## Related Documentation

- [Troubleshooting Guide](troubleshooting-guide.md)
- [API Endpoints Reference](api-endpoints-reference.md)
- [Hooks Reference](hooks-reference.md)
