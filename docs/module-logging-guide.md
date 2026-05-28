# WHMCS Module Logging Guide

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

This guide covers logging best practices for WHMCS modules, including when to log, what to log, log levels, sensitive data handling, and log analysis.

---

## WHMCS Logging Functions

### logModuleCall

```php
/**
 * Log module function calls
 * 
 * @param string $module Module name
 * @param string $action Action being performed
 * @param array $request Request data
 * @param string $response Response data
 * @param string $data Additional data
 * @param array $maskFields Fields to mask
 */
function logModuleCall(
    'your_module',
    'process_payment',
    [
        'invoice_id' => 123,
        'amount'     => 100.00,
        'currency'   => 'USD',
    ],
    json_encode($result),
    null,
    ['api_key', 'api_secret']
);
```

### logActivity

```php
/**
 * Log to WHMCS activity log
 * 
 * @param string $description Activity description
 * @param int $userId User ID (optional)
 */
logActivity("Your module synced data for client ID: 123", 123);

// Or without user association
logActivity("Your module: Cron job completed");
```

### Custom File Logging

```php
/**
 * Write to custom log file
 * 
 * @param string $message Log message
 * @param string $level Log level
 * @param array $context Additional context
 */
function your_module_log($message, $level = 'info', $context = [])
{
    $logFile = ROOTDIR . '/data/your_module.log';
    $timestamp = date('Y-m-d H:i:s');
    
    $entry = [
        'timestamp' => $timestamp,
        'level'     => strtoupper($level),
        'message'   => $message,
        'context'   => $context,
    ];
    
    $line = json_encode($entry) . "\n";
    
    // Ensure directory exists
    $dir = dirname($logFile);
    if (!is_dir($dir)) {
        mkdir($dir, 0755, true);
    }
    
    file_put_contents($logFile, $line, FILE_APPEND);
}
```

## Log Levels

### Log Level Definitions

| Level | Usage | Example |
|-------|-------|---------|
| DEBUG | Detailed debugging information | Function entry/exit, variable values |
| INFO | Normal operations | Successful API calls, data sync |
| WARNING | Recoverable issues | Rate limit approaching, cache miss |
| ERROR | Operation failures | API errors, database failures |
| CRITICAL | System failures | Module crash, complete failure |

### Level-Based Logging

```php
/**
 * Log with appropriate level
 */
function your_module_process($data)
{
    your_module_log('Starting process', 'debug', ['data_size' => strlen(json_encode($data))]);
    
    try {
        $result = your_module_apiCall($data);
        
        if ($result['success']) {
            your_module_log('Process completed', 'info', ['result_id' => $result['id']]);
        } else {
            your_module_log('Process failed', 'warning', $result);
        }
        
    } catch (\Exception $e) {
        your_module_log('Process error', 'error', [
            'error' => $e->getMessage(),
            'trace' => $e->getTraceAsString(),
        ]);
        
        throw $e;
    }
}
```

## What to Log

### Log These Events

```php
/**
 * Module operations to log
 */

// 1. Module activation/deactivation
logModuleCall('your_module', 'activate', [], 'Success');
logModuleCall('your_module', 'deactivate', [], 'Cleanup complete');

// 2. Configuration changes
logModuleCall('your_module', 'config_update', [
    'changed_fields' => ['api_key', 'test_mode'],
], 'Configuration updated');

// 3. API requests (within reason)
logModuleCall('your_module', 'api_request', [
    'endpoint' => '/clients',
    'method'   => 'POST',
], 'Response received', null, ['api_key', 'api_secret']);

// 4. User actions
logModuleCall('your_module', 'user_action', [
    'user_id'   => $_SESSION['uid'],
    'action'    => 'export_data',
    'format'    => 'csv',
], 'Export completed');

// 5. Cron job execution
logModuleCall('your_module', 'cron', [
    'start_time' => $startTime,
    'end_time'   => microtime(true),
    'processed'  => $processed,
    'failed'     => $failed,
], 'Cron completed');
```

### Do NOT Log

```php
/**
 * Avoid logging these
 */

// 1. Passwords - NEVER
$password = $_POST['password'];
// logModuleCall() - Never log $password!

// 2. Full API credentials
$apiKey = $params['apiKey'];
$apiSecret = $params['apiSecret'];
// Mask before logging!

// 3. Credit card numbers
$cardNumber = $_POST['card_num'];
// Never log!

// 4. Social Security Numbers
// Never log!

// 5. Excessive data
// Don't log entire POST arrays blindly
```

## Sensitive Data Handling

### Masking Helper

```php
/**
 * Mask sensitive fields before logging
 * 
 * @param array $data Data to log
 * @param array $maskFields Fields to mask
 * @return array Masked data
 */
function your_module_maskData($data, $maskFields = [])
{
    $defaultMasks = [
        'password', 'passwd',
        'api_key', 'apiKey', 'api_secret', 'apiSecret',
        'token', 'access_token', 'refresh_token',
        'secret', 'client_secret',
        'cvv', 'card_cvc',
        'ssn', 'social_security',
        'credit_card', 'card_number', 'pan',
        'pin',
    ];
    
    $masks = array_unique(array_merge($defaultMasks, $maskFields));
    
    $masked = [];
    foreach ($data as $key => $value) {
        $lowerKey = strtolower($key);
        
        $isSensitive = false;
        foreach ($masks as $mask) {
            if (strpos($lowerKey, strtolower($mask)) !== false) {
                $isSensitive = true;
                break;
            }
        }
        
        if ($isSensitive) {
            $masked[$key] = str_repeat('*', 8);
        } elseif (is_array($value)) {
            $masked[$key] = your_module_maskData($value, $maskFields);
        } else {
            $masked[$key] = $value;
        }
    }
    
    return $masked;
}
```

### Complete Logging Example

```php
/**
 * Complete logging with masking
 */
function your_module_logWithMasking($action, $request, $response, $additionalData = [])
{
    $maskedRequest = your_module_maskData($request);
    $maskedAdditional = your_module_maskData($additionalData);
    
    logModuleCall(
        'your_module',
        $action,
        $maskedRequest,
        is_string($response) ? $response : json_encode($response),
        $maskedAdditional
    );
}
```

## Database Audit Logging

### Audit Log Table

```php
/**
 * Create audit log table
 */
function your_module_createAuditTable()
{
    $sql = "CREATE TABLE IF NOT EXISTS `mod_your_audit` (
        `id` BIGINT(20) NOT NULL AUTO_INCREMENT PRIMARY KEY,
        `user_id` INT(10) NULL COMMENT 'User performing action (null for system)',
        `action` VARCHAR(50) NOT NULL,
        `entity_type` VARCHAR(50) NULL COMMENT 'Type of entity being acted upon',
        `entity_id` INT(10) NULL COMMENT 'ID of entity',
        `old_value` TEXT NULL COMMENT 'Previous value (JSON)',
        `new_value` TEXT NULL COMMENT 'New value (JSON)',
        `ip_address` VARCHAR(45) NULL,
        `user_agent` VARCHAR(255) NULL,
        `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
        INDEX `idx_user_id` (`user_id`),
        INDEX `idx_action` (`action`),
        INDEX `idx_entity` (`entity_type`, `entity_id`),
        INDEX `idx_created` (`created_at`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
    
    full_query($sql);
}

/**
 * Log audit entry
 */
function your_module_audit($action, $entityType = null, $entityId = null, $oldValue = null, $newValue = null)
{
    $userId = $_SESSION['uid'] ?? null;
    $ipAddress = $_SERVER['REMOTE_ADDR'] ?? null;
    $userAgent = $_SERVER['HTTP_USER_AGENT'] ?? null;
    
    insert_query('mod_your_audit', [
        'user_id'    => $userId,
        'action'     => $action,
        'entity_type' => $entityType,
        'entity_id'   => $entityId,
        'old_value'   => $oldValue ? json_encode($oldValue) : null,
        'new_value'   => $newValue ? json_encode($newValue) : null,
        'ip_address'  => $ipAddress,
        'user_agent'  => substr($userAgent, 0, 255),
        'created_at'  => date('Y-m-d H:i:s'),
    ]);
}

// Usage
$oldSettings = get_config_var('your_settings');
your_module_audit('settings_updated', 'config', null, $oldSettings, $_POST);

// Log deletions with before/after
$record = WHMCS\Database\Capsule::table('mod_your_records')
    ->where('id', $recordId)
    ->first();
your_module_audit('record_deleted', 'record', $recordId, $record, null);
```

## Log Analysis

### Log Rotation

```php
/**
 * Rotate logs when they get too large
 */
function your_module_rotateLogs($maxSize = 10485760, $keepCount = 5)
{
    $logFile = ROOTDIR . '/data/your_module.log';
    
    if (!file_exists($logFile)) {
        return;
    }
    
    $size = filesize($logFile);
    
    if ($size < $maxSize) {
        return;
    }
    
    // Archive current log
    $archiveName = $logFile . '.' . date('Y-m-d-His') . '.gz';
    
    $fp = gzopen($archiveName, 'w9');
    gzwrite($fp, file_get_contents($logFile));
    gzclose($fp);
    
    // Clear current log
    file_put_contents($logFile, '');
    
    // Remove old archives
    $logs = glob($logFile . '.*.gz');
    usort($logs, function($a, $b) { return strcmp($b, $a); });
    
    foreach (array_slice($logs, $keepCount) as $oldLog) {
        @unlink($oldLog);
    }
}
```

### Log Parsing

```php
/**
 * Parse log file
 */
function your_module_parseLogs($logFile, $options = [])
{
    $limit = $options['limit'] ?? 100;
    $level = $options['level'] ?? null;
    $search = $options['search'] ?? null;
    $since = $options['since'] ?? null;
    
    $handle = fopen($logFile, 'r');
    $logs = [];
    
    while (!feof($handle) && count($logs) < $limit) {
        $line = fgets($handle);
        
        if (empty($line)) {
            continue;
        }
        
        $entry = json_decode($line, true);
        
        if (!$entry) {
            continue;
        }
        
        // Filter by level
        if ($level && ($entry['level'] ?? '') !== strtoupper($level)) {
            continue;
        }
        
        // Filter by search
        if ($search && stripos($entry['message'], $search) === false) {
            continue;
        }
        
        // Filter by time
        if ($since && strtotime($entry['timestamp']) < strtotime($since)) {
            continue;
        }
        
        $logs[] = $entry;
    }
    
    fclose($handle);
    
    return $logs;
}
```

## Debug Logging

### Debug Mode Implementation

```php
/**
 * Debug logging (only when enabled)
 */
function your_module_debug($message, $context = [])
{
    $debugEnabled = get_config_var('your_module_debug') || 
                    $_SESSION['adminid'] && $_SESSION['debug_mode'] ?? false;
    
    if (!$debugEnabled) {
        return;
    }
    
    your_module_log($message, 'debug', $context);
}

/**
 * Conditional debug info
 */
function your_module_debugIf($condition, $message, $context = [])
{
    if ($condition) {
        your_module_debug($message, $context);
    }
}

// Usage
your_module_debug('Function parameters', $params);
your_module_debugIf(count($debugLog) > 1000, 'Large dataset warning', ['count' => count($debugLog)]);
```

---

## Related Skills and Workflows

- `module-error-handling-guide` - Error handling with logging
- `module-security-standards` - Security logging requirements
- `security-audit-checklist` - Audit logging requirements
- `performance-optimization` - Performance logging
