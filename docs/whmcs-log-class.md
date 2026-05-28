# WHMCS Log Class Reference

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-logging`, `whmcs-audit-log`, `whmcs-error-handling`

---

## Overview

WHMCS provides multiple logging mechanisms for tracking system activities, debugging, and audit trails. This reference covers the core logging classes and best practices for module development.

---

## Activity Log (`logActivity`)

The primary method for logging admin activities to the WHMCS activity log (`tblactivitylog`).

### Basic Usage

```php
<?php
// Simple message
logActivity("Module action completed successfully");

// With context
logActivity("Custom Module: Service #{$serviceId} created for User #{$userId}");
```

### Activity Log Function Signature

```php
/**
 * Log activity to the WHMCS activity log
 *
 * @param string     $message     The log message
 * @param int|null  $userId      User ID (optional, defaults to admin)
 * @param string     $description Additional description
 * @return void
 */
logActivity(string $message, int $userId = null, string $description = ''): void;
```

### Examples

```php
// Log service creation
add_hook('AfterModuleCreate', 1, function($vars) {
    logActivity("Provisioning Module: Server created - Service #{$vars['serviceid']}");
});

// Log client login
add_hook('ClientLogin', 1, function($vars) {
    logActivity("Client Login: User #{$vars['userid']} logged in from IP {$_SERVER['REMOTE_ADDR']}");
});

// Log payment
add_hook('InvoicePaid', 1, function($vars) {
    logActivity("Payment received: Invoice #{$vars['invoiceid']}, Amount: {$vars['amount']}");
});
```

---

## Transaction Log (`logTransaction`)

Used specifically for recording payment gateway transactions.

### Function Signature

```php
/**
 * Log a payment gateway transaction
 *
 * @param string $gateway       Gateway name
 * @param array $data         Transaction data
 * @param string $status      Transaction status (Success, Failed, etc.)
 * @param string $note        Optional note
 * @return void
 */
logTransaction(string $gateway, array $data, string $status, string $note = ''): void;
```

### Examples

```php
// Standard gateway transaction logging
function mygateway_link(array $params): string {
    $transactionData = [
        'amount' => $params['amount'],
        'currency' => $params['currency'],
        'invoice_id' => $params['invoiceid'],
        'client_email' => $params['client']['email'],
    ];

    // Log initiation
    logTransaction('MyGateway', $transactionData, 'Submitted');

    return $formHtml;
}

// Callback handler logging
function mygateway_callback() {
    $response = $_POST;

    if ($response['status'] === 'success') {
        logTransaction('MyGateway', $response, 'Success', 'Payment completed');
        addInvoicePayment($response['invoice_id'], $response['transaction_id'], $response['amount'], 0, 'MyGateway');
    } else {
        logTransaction('MyGateway', $response, 'Failed', $response['error_message']);
    }
}
```

---

## Module Call Logging (`logModuleCall`)

Records API calls and responses for debugging and audit purposes.

### Function Signature

```php
/**
 * Log module API call
 *
 * @param string $module      Module name
 * @param string $action     Action performed
 * @param mixed  $request    Request data
 * @param mixed  $response   Response data
 * @param string $postData   Posted data (for gateway callbacks)
 * @param array  $redact     Sensitive fields to mask
 * @return void
 */
logModuleCall(
    string $module,
    string $action,
    mixed $request,
    mixed $response,
    string $postData = null,
    array $redact = []
): void;
```

### Examples

```php
<?php
// Log API request/response
function callExternalApi(array $params): array {
    $api = new MyApiClient($params['api_key']);

    logModuleCall(
        'mymodule',
        'API Call',
        ['endpoint' => '/servers', 'method' => 'POST'],
        ['server_id' => 123, 'status' => 'active'],
        null,
        ['api_key']
    );

    return $api->createServer($params);
}

// Log with sensitive data redaction
add_hook('AfterModuleCreate', 1, function($vars) {
    logModuleCall(
        'ProvisioningModule',
        'CreateAccount',
        $vars,
        ['result' => 'success'],
        null,
        ['server_password', 'root_password']
    );
});
```

---

## Admin Log Class

Advanced logging to the WHMCS administrative log system.

### Class Reference

```php
<?php
use WHMCS\Log\Activity;

/**
 * Log admin activity
 */
Activity::log($message, $adminId = null, $description = null);

// Or direct usage
$log = new \WHMCS\Log\Activity();
$log->log("Custom message");

// Log with custom severity
$log->log("Important action", null, [
    'severity' => 'warning',
    'category' => 'billing',
]);
```

---

## Custom Module Logging

### Module Log Table Setup

```php
<?php
/**
 * Create module-specific log table
 */
function setupModuleLogging(): void {
    Capsule::schema()->create('mod_mymodule_logs', function($t) {
        $t->increments('id');
        $t->string('level', 20);        // debug, info, warning, error
        $t->string('category', 50);    // api, user, system
        $t->text('message');
        $t->json('context')->nullable();
        $t->string('ip_address', 45)->nullable();
        $t->integer('user_id')->unsigned()->nullable();
        $t->integer('admin_id')->unsigned()->nullable();
        $t->timestamp('created_at')->useCurrent();

        $t->index(['level', 'created_at']);
        $t->index(['category']);
        $t->index(['user_id']);
    });
}
```

### Module Logger Implementation

```php
<?php
/**
 * Module-specific logging class
 */
class ModuleLogger {
    const LEVEL_DEBUG = 'debug';
    const LEVEL_INFO = 'info';
    const LEVEL_WARNING = 'warning';
    const LEVEL_ERROR = 'error';

    private string $module;
    private bool $debugEnabled;

    public function __construct(string $module, bool $debugEnabled = false) {
        $this->module = $module;
        $this->debugEnabled = $debugEnabled;
    }

    /**
     * Write log entry
     */
    public function log(string $level, string $message, array $context = []): void {
        // Also log to WHMCS activity log for errors
        if ($level === self::LEVEL_ERROR) {
            logActivity("[{$this->module}] ERROR: " . $message);
        }

        // Write to module log table
        Capsule::table('mod_' . $this->module . '_logs')->insert([
            'level' => $level,
            'category' => $context['category'] ?? 'general',
            'message' => $message,
            'context' => json_encode($context),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null,
            'user_id' => $_SESSION['uid'] ?? null,
            'admin_id' => $_SESSION['adminid'] ?? null,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function debug(string $message, array $context = []): void {
        if ($this->debugEnabled) {
            $this->log(self::LEVEL_DEBUG, $message, $context);
        }
    }

    public function info(string $message, array $context = []): void {
        $this->log(self::LEVEL_INFO, $message, $context);
    }

    public function warning(string $message, array $context = []): void {
        $this->log(self::LEVEL_WARNING, $message, $context);
    }

    public function error(string $message, array $context = []): void {
        $this->log(self::LEVEL_ERROR, $message, $context);
    }

    /**
     * Log API call
     */
    public function apiCall(string $endpoint, array $request, array $response, float $durationMs): void {
        $this->info('API call: ' . $endpoint, [
            'category' => 'api',
            'endpoint' => $endpoint,
            'duration_ms' => round($durationMs, 2),
            'response_code' => $response['code'] ?? 0,
            'request' => $this->sanitizeForLog($request),
        ]);
    }

    /**
     * Sanitize sensitive data for logging
     */
    private function sanitizeForLog(array $data): array {
        $sensitive = ['password', 'api_key', 'secret', 'token', 'credit_card'];

        foreach ($sensitive as $key) {
            if (isset($data[$key])) {
                $data[$key] = '***REDACTED***';
            }
        }

        return $data;
    }
}
```

### Usage Examples

```php
<?php
// In provisioning module
$logger = new ModuleLogger('mymodule', true);

function mymodule_CreateAccount(array $params): string {
    global $logger;

    $logger->info('Creating account', [
        'service_id' => $params['serviceid'],
        'username' => $params['username'],
    ]);

    try {
        $startTime = microtime(true);
        $result = $api->createServer($params);
        $duration = (microtime(true) - $startTime) * 1000;

        $logger->apiCall('/servers', $params, $result, $duration);
        $logger->info('Account created successfully');

        return 'success';

    } catch (\Exception $e) {
        $logger->error('Account creation failed', ['error' => $e->getMessage()]);
        return 'Error: ' . $e->getMessage();
    }
}
```

---

## Error Logging

### WHMCS Error Log

```php
<?php
use WHMCS\Utility\ErrorLog;

// Log errors to WHMCS error log
$errorLog = new ErrorLog();
$errorLog->log($message, $level = ErrorLog::SEVERITY_ERROR);
```

### PHP Error Logging

```php
<?php
// Custom error handler
set_error_handler(function($severity, $message, $file, $line) {
    if (!(error_reporting() & $severity)) {
        return false;
    }

    logActivity("PHP Error: {$message} in {$file} on line {$line}");

    return false; // Let PHP handle it too
});

// Log exceptions
set_exception_handler(function(\Throwable $e) {
    logActivity("Uncaught Exception: " . $e->getMessage() . "\n" . $e->getTraceAsString());

    // Display user-friendly message
    echo "An error occurred. Please contact support.";
});
```

---

## Log Viewer Implementation

### Admin Log Viewer

```php
<?php
/**
 * View module logs in admin area
 */
function viewModuleLogs(array $filters = [], int $limit = 100, int $offset = 0): array {
    $query = Capsule::table('mod_mymodule_logs');

    if (!empty($filters['level'])) {
        $query->where('level', $filters['level']);
    }

    if (!empty($filters['category'])) {
        $query->where('category', $filters['category']);
    }

    if (!empty($filters['from_date'])) {
        $query->where('created_at', '>=', $filters['from_date']);
    }

    if (!empty($filters['to_date'])) {
        $query->where('created_at', '<=', $filters['to_date']);
    }

    if (!empty($filters['user_id'])) {
        $query->where('user_id', $filters['user_id']);
    }

    return [
        'logs' => $query->orderBy('created_at', 'desc')->offset($offset)->limit($limit)->get(),
        'total' => $query->count(),
    ];
}
```

---

## Log Retention

### Cleanup Hook

```php
<?php
/**
 * Clean old logs periodically
 */
add_hook('DailyCronJob', 1, function() {
    $retentionDays = 90;

    Capsule::table('mod_mymodule_logs')
        ->where('created_at', '<', date('Y-m-d H:i:s', strtotime("-{$retentionDays} days")))
        ->where('level', '<>', 'error')  // Keep errors longer
        ->delete();
});
```

---

## Log Levels Reference

| Level | Usage | Retention Suggestion |
|-------|-------|-------------------|
| `debug` | Detailed debugging information | Development only |
| `info` | Normal operational events | 30 days |
| `warning` | Potential issues requiring attention | 60 days |
| `error` | Errors that need investigation | 90+ days |

---

## Best Practices

1. **Log all module actions** - Track provisioning, termination, etc.
2. **Include context** - Service ID, user ID, transaction ID
3. **Sanitize sensitive data** - Never log passwords or API keys
4. **Set appropriate levels** - debug for development, error for errors
5. **Implement log rotation** - Clean up old logs automatically
6. **Create audit trails** - Log all admin actions
7. **Include timestamps** - All logs should have creation time
8. **Log API calls** - Track external API requests/responses
9. **Monitor error logs** - Set up alerts for error log entries

---

## Related Documentation

- [Logging Skill](../skills/whmcs-logging)
- [Audit Log Skill](../skills/whmcs-audit-log)
- [Error Handling Guide](api-error-handling.md)
