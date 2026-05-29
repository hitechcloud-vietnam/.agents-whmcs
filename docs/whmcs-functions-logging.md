# WHMCS Logging Functions

Complete reference for WHMCS logging and debugging functions.

## logActivity()

Logs an activity entry to the activity log table.

```php
/**
 * Log an activity to the activity log
 * 
 * @param string $description The activity description
 * @param int|null $userId The user ID (null for client, 0 for admin system)
 * @return bool Success status
 */
logActivity(string $description, int|null $userId = null): bool
```

**Example:**
```php
// Log as admin (userId = 0)
logActivity("Invoice #1234 marked as paid", 0);

// Log as specific admin
logActivity("Updated client pricing tier", 15);

// Log client activity
logActivity("Client upgraded service package", $clientId);
```

## logAdminActivity()

Logs an admin activity with module context.

```php
/**
 * Log an admin activity with full context
 * 
 * @param string $description Activity description
 * @param string $module Module name (optional)
 * @param string $action Action performed (optional)
 * @param int $adminId Admin user ID
 * @param string $ipAddress Client IP address
 * @return bool Success status
 */
logAdminActivity(
    string $description,
    string $module = '',
    string $action = '',
    int $adminId = 0,
    string $ipAddress = ''
): bool
```

**Example:**
```php
logAdminActivity(
    "Module configuration updated for provisioning module",
    "provisioning",
    "configure",
    $_SESSION['adminid'],
    $_SERVER['REMOTE_ADDR']
);
```

## logClientActivity()

Logs activity for client-side actions.

```php
/**
 * Log client activity
 * 
 * @param string $description Activity description
 * @param int $clientId Client user ID
 * @return bool Success status
 */
logClientActivity(string $description, int $clientId): bool
```

**Example:**
```php
logClientActivity("Downloaded invoice PDF", $clientId);
logClientActivity("Updated contact information", $clientId);
```

## logModuleCall()

Logs module function calls for debugging.

```php
/**
 * Log a module function call
 * 
 * @param string $module Module name
 * @param string $action Action called
 * @param string $request Request data (serialized)
 * @param string $response Response data (serialized)
 * @param string $status Success/failure status
 * @param string $gatewayToken Gateway token (optional)
 * @param float $startTime Start timestamp
 * @param float $endTime End timestamp
 * @return bool Success status
 */
logModuleCall(
    string $module,
    string $action,
    string $request,
    string $response,
    string $status,
    string $gatewayToken = '',
    float $startTime = 0,
    float $endTime = 0
): bool
```

**Example:**
```php
$startTime = microtime(true);

try {
    $response = $module->createInstance($params);
    $endTime = microtime(true);
    
    logModuleCall(
        'custommodule',
        'CreateInstance',
        serialize($params),
        serialize($response),
        'Success',
        '',
        $startTime,
        $endTime
    );
} catch (Exception $e) {
    logModuleCall(
        'custommodule',
        'CreateInstance',
        serialize($params),
        $e->getMessage(),
        'Failed',
        '',
        $startTime,
        microtime(true)
    );
}
```

## logTicketActivity()

Logs activity related to support tickets.

```php
/**
 * Log ticket activity
 * 
 * @param int $ticketId Ticket ID
 * @param string $action Action performed
 * @param string $description Description of the action
 * @param int|null $adminId Admin who performed the action
 * @return bool Success status
 */
logTicketActivity(
    int $ticketId,
    string $action,
    string $description,
    int|null $adminId = null
): bool
```

**Example:**
```php
logTicketActivity($ticketId, 'status_change', 'Status changed to in_progress');
logTicketActivity($ticketId, 'priority_change', 'Priority escalated to high');
```

## debug()

Outputs debug information when debug mode is enabled.

```php
/**
 * Output debug information
 * 
 * @param mixed $var Variable to debug
 * @param string $label Optional label
 * @param bool $force Force output regardless of debug setting
 * @return void
 */
debug(mixed $var, string $label = '', bool $force = false): void
```

**Example:**
```php
// Basic debug
debug($orderData);

// Labeled debug
debug($invoice, 'Invoice Data before processing');

// Force output even when debug is off
debug($criticalVar, 'Critical Value', true);
```

## logEvent()

Logs custom events for monitoring.

```php
/**
 * Log a custom event
 * 
 * @param string $type Event type
 * @param array $data Event data
 * @param string $severity Severity level (info, warning, error)
 * @return bool Success status
 */
logEvent(string $type, array $data, string $severity = 'info'): bool
```

**Example:**
```php
logEvent('custom_action', [
    'order_id' => 123,
    'amount' => 99.99,
    'gateway' => 'stripe'
], 'info');

logEvent('error_condition', [
    'error_code' => 'E1001',
    'error_message' => 'Gateway timeout'
], 'error');
```

## Monitoring Functions

### getActivityLog()

Retrieves activity log entries with filtering.

```php
/**
 * Get activity log entries
 * 
 * @param array $filters Filter options
 * @param int $limit Number of records
 * @param int $offset Starting offset
 * @return array Activity log entries
 */
function getActivityLog(array $filters = [], int $limit = 100, int $offset = 0): array
{
    $query = "SELECT * FROM tblactivitylog WHERE 1=1";
    $params = [];
    
    if (!empty($filters['userId'])) {
        $query .= " AND userid = ?";
        $params[] = $filters['userId'];
    }
    
    if (!empty($filters['dateFrom'])) {
        $query .= " AND date >= ?";
        $params[] = $filters['dateFrom'];
    }
    
    if (!empty($filters['dateTo'])) {
        $query .= " AND date <= ?";
        $params[] = $filters['dateTo'];
    }
    
    $query .= " ORDER BY id DESC LIMIT ? OFFSET ?";
    $params[] = $limit;
    $params[] = $offset;
    
    return Capsule::select($query, $params);
}
```

**Example:**
```php
$logs = getActivityLog([
    'userId' => 15,
    'dateFrom' => '2024-01-01',
    'dateTo' => '2024-12-31'
], 50, 0);
```

### clearActivityLog()

Clears old activity log entries.

```php
/**
 * Clear old activity log entries
 * 
 * @param int $daysOld Remove entries older than this many days
 * @return int Number of entries deleted
 */
function clearActivityLog(int $daysOld = 90): int
{
    $cutoffDate = date('Y-m-d H:i:s', strtotime("-{$daysOld} days"));
    
    return Capsule::table('tblactivitylog')
        ->where('date', '<', $cutoffDate)
        ->delete();
}
```

**Example:**
```php
// Clear logs older than 30 days
$deleted = clearActivityLog(30);
```

## Best Practices

1. **Use appropriate log levels** - Use 'info' for routine actions, 'warning' for recoverable issues, and 'error' for failures
2. **Include relevant context** - Log enough data to reconstruct events
3. **Sanitize sensitive data** - Never log passwords, full credit card numbers, or API keys
4. **Use structured data** - Pass arrays instead of concatenated strings
5. **Consider performance** - Avoid logging in tight loops; batch when possible
6. **Rotate logs regularly** - Implement log rotation to prevent disk space issues

## Error Logging

```php
/**
 * Custom error handler that logs to WHMCS activity log
 */
function whmcsErrorHandler($errno, $errstr, $errfile, $errline)
{
    $errorTypes = [
        E_ERROR => 'Error',
        E_WARNING => 'Warning',
        E_PARSE => 'Parse Error',
        E_NOTICE => 'Notice',
        E_CORE_ERROR => 'Core Error',
        E_CORE_WARNING => 'Core Warning',
        E_COMPILE_ERROR => 'Compile Error',
        E_COMPILE_WARNING => 'Compile Warning',
        E_USER_ERROR => 'User Error',
        E_USER_WARNING => 'User Warning',
        E_USER_NOTICE => 'User Notice',
        E_STRICT => 'Strict Notice',
        E_RECOVERABLE_ERROR => 'Recoverable Error',
        E_DEPRECATED => 'Deprecated',
        E_USER_DEPRECATED => 'User Deprecated'
    ];
    
    $type = $errorTypes[$errno] ?? 'Unknown';
    $message = "[{$type}] {$errstr} in {$errfile} on line {$errline}";
    
    logActivity($message, 0);
    
    return false;
}

set_error_handler('whmcsErrorHandler');
```

## Related Functions

- [whmcs-functions-transactions.md](whmcs-functions-transactions.md) - Transaction logging
- [whmcs-functions-adminlog.md](whmcs-schema-adminlog.md) - Admin activity schema