# WHMCS Audit Logging Workflow

## Overview
This workflow covers implementing comprehensive audit logging for compliance and security monitoring.

## Step 1: Audit Logging Service

```php
<?php
// src/Service/AuditLogService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class AuditLogService
{
    private $logTable = 'mod_audit_logs';
    private $sensitiveFields = ['password', 'credit_card', 'ssn', 'api_key', 'secret'];

    public function log(string $action, string $category, array $data = [], int $userId = null): int
    {
        // Sanitize sensitive data
        $data = $this->sanitizeData($data);

        $userId = $userId ?? ($_SESSION['adminid'] ?? null);
        $ipAddress = $_SERVER['REMOTE_ADDR'] ?? 'CLI';
        $userAgent = $_SERVER['HTTP_USER_AGENT'] ?? '';

        return Capsule::table($this->logTable)->insertGetId([
            'action' => $action,
            'category' => $category,
            'user_id' => $userId,
            'user_type' => $userId ? 'admin' : 'system',
            'ip_address' => $ipAddress,
            'user_agent' => substr($userAgent, 0, 500),
            'data' => json_encode($data),
            'result' => $data['result'] ?? 'success',
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    public function logLogin(int $userId, string $result, string $method = 'password'): void
    {
        $this->log('login', 'authentication', [
            'result' => $result,
            'method' => $method
        ], $userId);
    }

    public function logLogout(int $userId): void
    {
        $this->log('logout', 'authentication', [], $userId);
    }

    public function logDataAccess(string $resource, int $resourceId, string $action): void
    {
        $this->log($action, 'data_access', [
            'resource' => $resource,
            'resource_id' => $resourceId
        ]);
    }

    public function logConfigChange(string $setting, $oldValue, $newValue): void
    {
        $this->log('update', 'configuration', [
            'setting' => $setting,
            'old_value' => $oldValue,
            'new_value' => $newValue
        ]);
    }

    public function logPayment(int $invoiceId, string $action, array $paymentData): void
    {
        $this->log($action, 'payment', array_merge([
            'invoice_id' => $invoiceId
        ], $paymentData));
    }

    public function logApiAccess(string $endpoint, string $method, int $responseCode): void
    {
        $this->log('api_call', 'api', [
            'endpoint' => $endpoint,
            'method' => $method,
            'response_code' => $responseCode
        ]);
    }

    public function queryLogs(array $filters = [], int $limit = 100): array
    {
        $query = Capsule::table($this->logTable);

        if (!empty($filters['action'])) {
            $query->where('action', $filters['action']);
        }

        if (!empty($filters['category'])) {
            $query->where('category', $filters['category']);
        }

        if (!empty($filters['user_id'])) {
            $query->where('user_id', $filters['user_id']);
        }

        if (!empty($filters['date_from'])) {
            $query->where('created_at', '>=', $filters['date_from']);
        }

        if (!empty($filters['date_to'])) {
            $query->where('created_at', '<=', $filters['date_to']);
        }

        if (!empty($filters['ip_address'])) {
            $query->where('ip_address', $filters['ip_address']);
        }

        return $query
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get()
            ->toArray();
    }

    public function getFailedLogins(int $hours = 24): array
    {
        return Capsule::table($this->logTable)
            ->where('action', 'login')
            ->where('data', 'like', '%"result":"failed"%')
            ->where('created_at', '>=', date('Y-m-d H:i:s', strtotime("-{$hours} hours")))
            ->orderBy('created_at', 'desc')
            ->get()
            ->toArray();
    }

    public function getSuspiciousActivity(int $hours = 24): array
    {
        return Capsule::table($this->logTable)
            ->where('created_at', '>=', date('Y-m-d H:i:s', strtotime("-{$hours} hours")))
            ->where(function($query) {
                $query->where('result', 'failed')
                    ->orWhere('action', 'like', 'delete%')
                    ->orWhere('action', 'like', 'export%');
            })
            ->get()
            ->toArray();
    }

    private function sanitizeData(array $data): array
    {
        foreach ($data as $key => $value) {
            $keyLower = strtolower($key);

            foreach ($this->sensitiveFields as $field) {
                if (strpos($keyLower, $field) !== false) {
                    $data[$key] = '[REDACTED]';
                    break;
                }
            }
        }

        return $data;
    }
}
```

## Step 2: Audit Logging Hook

```php
<?php
// includes/hooks/audit_hook.php

use WHMCS\Module\Addon\YourModule\Service\AuditLogService;

$auditService = new AuditLogService();

// Log admin logins
add_hook('AdminAreaPage', 1, function($params) {
    // Track page access
});

// Log failed login attempts
add_hook('LoginFailure', 1, function($params) {
    $auditService->logLogin($params['user_id'] ?? 0, 'failed', $params['method'] ?? 'unknown');
});

// Log successful logins
add_hook('AdminLogin', 1, function($params) {
    $auditService->logLogin($params['admin_id'], 'success');
});
```

## Verification Checklist

- [ ] Audit log service implemented
- [ ] Logging hooks registered
- [ ] Sensitive data masking working
- [ ] Log querying working
- [ ] Failed login tracking working
- [ ] Suspicious activity detection working
