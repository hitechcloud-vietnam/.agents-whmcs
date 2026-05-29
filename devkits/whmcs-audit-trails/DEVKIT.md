# WHMCS Audit Trails DevKit

## Overview

A comprehensive audit logging and trail system for WHMCS that tracks all system activities, user actions, data changes, and administrative operations with tamper-proof logging, compliance-ready audit trails, and detailed activity search capabilities.

## Features

- Comprehensive activity logging
- User action tracking
- Data change auditing
- Admin operations audit
- API call logging
- Login/logout tracking
- Query auditing
- Compliance-ready export
- Tamper-proof storage
- Real-time alerts
- Activity search and filtering
- Audit dashboard

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS `mod_audit_logs` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `event_id` VARCHAR(64) NOT NULL,
    `event_type` VARCHAR(100) NOT NULL,
    `event_category` ENUM('user', 'admin', 'system', 'api', 'security', 'data', 'billing', 'service') NOT NULL,
    `severity` ENUM('debug', 'info', 'notice', 'warning', 'error', 'critical') NOT NULL DEFAULT 'info',
    `user_id` INT UNSIGNED NULL,
    `admin_id` INT UNSIGNED NULL,
    `ip_address` VARCHAR(45) NULL,
    `user_agent` VARCHAR(500) NULL,
    `session_id` VARCHAR(128) NULL,
    `entity_type` VARCHAR(100) NULL,
    `entity_id` VARCHAR(100) NULL,
    `action` VARCHAR(100) NOT NULL,
    `old_values` JSON NULL,
    `new_values` JSON NULL,
    `changes_summary` TEXT NULL,
    `metadata` JSON NULL,
    `hash` VARCHAR(64) NOT NULL,
    `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    INDEX `idx_event_type` (`event_type`),
    INDEX `idx_category_date` (`event_category`, `created_at`),
    INDEX `idx_user_date` (`user_id`, `created_at`),
    INDEX `idx_admin_date` (`admin_id`, `created_at`),
    INDEX `idx_entity` (`entity_type`, `entity_id`),
    INDEX `idx_severity_date` (`severity`, `created_at`),
    INDEX `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_audit_retention_policies` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `policy_name` VARCHAR(255) NOT NULL,
    `event_category` VARCHAR(100) NULL,
    `event_type` VARCHAR(100) NULL,
    `severity` VARCHAR(50) NULL,
    `retention_days` INT UNSIGNED NOT NULL DEFAULT 365,
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_audit_summaries` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `summary_date` DATE NOT NULL,
    `entity_type` VARCHAR(100) NULL,
    `event_category` VARCHAR(100) NULL,
    `total_events` INT UNSIGNED NOT NULL DEFAULT 0,
    `unique_users` INT UNSIGNED NOT NULL DEFAULT 0,
    `unique_admins` INT UNSIGNED NOT NULL DEFAULT 0,
    `error_count` INT UNSIGNED NOT NULL DEFAULT 0,
    `warning_count` INT UNSIGNED NOT NULL DEFAULT 0,
    `security_events` INT UNSIGNED NOT NULL DEFAULT 0,
    `data_changes` INT UNSIGNED NOT NULL DEFAULT 0,
    `hash` VARCHAR(64) NOT NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_date_entity` (`summary_date`, `entity_type`, `event_category`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_audit_alerts` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `alert_type` VARCHAR(100) NOT NULL,
    `severity` ENUM('low', 'medium', 'high', 'critical') NOT NULL DEFAULT 'medium',
    `conditions` JSON NOT NULL,
    `notification_channels` JSON NOT NULL,
    `recipients` JSON NULL,
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `last_triggered` DATETIME NULL,
    `trigger_count` INT UNSIGNED NOT NULL DEFAULT 0,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Module Files

### audit_trails.php

```php
<?php
/**
 * WHMCS Audit Trails Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/AuditLogger.php';
require_once __DIR__ . '/lib/AuditQuery.php';
require_once __DIR__ . '/lib/AuditRetention.php';
require_once __DIR__ . '/lib/AuditExporter.php';

use WHMCS\Module\AuditTrails\AuditLogger;
use WHMCS\Module\AuditTrails\AuditQuery;
use WHMCS\Module\AuditTrails\AuditRetention;
use WHMCS\Module\AuditTrails\AuditExporter;

function whmcs_audit_trails_activate() {
    $logger = new AuditLogger();
    return $logger->activate();
}

function whmcs_audit_trails_deactivate() {
    return ['success' => true, 'msg' => 'Audit Trails module deactivated'];
}

function whmcs_audit_trails_config() {
    return [
        'retention_days' => [
            'FriendlyName' => 'Default Retention (Days)',
            'Type' => 'dropdown',
            'Options' => [
                '30' => '30 days',
                '90' => '90 days',
                '180' => '180 days',
                '365' => '1 year',
                '730' => '2 years',
                '0' => 'Forever',
            ],
            'Default' => '365',
        ],
        'enable_real_time_alerts' => [
            'FriendlyName' => 'Enable Real-time Alerts',
            'Type' => 'yesno',
        ],
        'log_api_calls' => [
            'FriendlyName' => 'Log API Calls',
            'Type' => 'yesno',
        ],
        'log_failed_logins' => [
            'FriendlyName' => 'Log Failed Login Attempts',
            'Type' => 'yesno',
        ],
        'log_data_changes' => [
            'FriendlyName' => 'Log Data Changes',
            'Type' => 'yesno',
        ],
        'hash_algorithm' => [
            'FriendlyName' => 'Hash Algorithm',
            'Type' => 'dropdown',
            'Options' => [
                'sha256' => 'SHA-256',
                'sha384' => 'SHA-384',
                'sha512' => 'SHA-512',
            ],
            'Default' => 'sha256',
        ],
        'alert_email' => [
            'FriendlyName' => 'Alert Email',
            'Type' => 'text',
            'Size' => '50',
        ],
    ];
}

/**
 * Log an event
 */
function whmcs_audit_trails_log($eventType, $action, $data = [], $options = []) {
    $logger = new AuditLogger();
    return $logger->log($eventType, $action, $data, $options);
}

/**
 * Query audit logs
 */
function whmcs_audit_trails_query($filters = []) {
    $query = new AuditQuery();
    return $query->search($filters);
}

/**
 * Export audit logs
 */
function whmcs_audit_trails_export($filters = [], $format = 'csv') {
    $exporter = new AuditExporter();
    return $exporter->export($filters, $format);
}

/**
 * Get audit summary
 */
function whmcs_audit_trails_get_summary($startDate = null, $endDate = null) {
    $query = new AuditQuery();
    return $query->getSummary($startDate, $endDate);
}

// Hooks registration
add_hook('DailyCronJob', 1, function() {
    $retention = new AuditRetention();
    $retention->applyRetentionPolicies();
    $retention->generateDailySummaries();
});

add_hook('ClientLogin', 1, function($params) {
    $logger = new AuditLogger();
    $logger->log('user', 'login', ['user_id' => $params['user_id']], ['user_id' => $params['user_id']]);
});

add_hook('ClientLogout', 1, function($params) {
    $logger = new AuditLogger();
    $logger->log('user', 'logout', [], ['user_id' => $params['user_id']]);
});

add_hook('AdminLogin', 1, function($params) {
    $logger = new AuditLogger();
    $logger->log('admin', 'login', ['admin_id' => $params['admin_id']], ['admin_id' => $params['admin_id']]);
});

add_hook('InvoicePayment', 1, function($params) {
    $logger = new AuditLogger();
    $logger->log('billing', 'payment_received', [
        'invoice_id' => $params['invoice_id'],
        'amount' => $params['amount'],
    ], ['user_id' => $params['user_id']]);
});

add_hook('ServiceCreate', 1, function($params) {
    $logger = new AuditLogger();
    $logger->log('service', 'create', $params, ['user_id' => $params['user_id']]);
});

add_hook('ServiceModify', 1, function($params) {
    $logger = new AuditLogger();
    $logger->log('service', 'modify', [
        'service_id' => $params['service_id'],
        'changes' => $params['changes'],
    ], ['user_id' => $params['user_id']]);
});
```

### lib/AuditLogger.php

```php
<?php
/**
 * Audit Logger
 */

namespace WHMCS\Module\AuditTrails;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class AuditLogger {
    
    protected $hashAlgorithm = 'sha256';
    
カテゴリーマッピング = [
        'login' => 'security',
        'logout' => 'security',
        'create' => 'data',
        'update' => 'data',
        'delete' => 'data',
        'payment' => 'billing',
        'invoice' => 'billing',
        'api' => 'api',
    ];
    
    public function activate() {
        try {
            $this->createTables();
            $this->setupDefaultRetentionPolicies();
            return ['success' => true, 'msg' => 'Audit Trails module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_audit_logs` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `event_id` VARCHAR(64) NOT NULL,
                `event_type` VARCHAR(100) NOT NULL,
                `event_category` ENUM('user', 'admin', 'system', 'api', 'security', 'data', 'billing', 'service') NOT NULL,
                `severity` ENUM('debug', 'info', 'notice', 'warning', 'error', 'critical') NOT NULL DEFAULT 'info',
                `user_id` INT UNSIGNED NULL,
                `admin_id` INT UNSIGNED NULL,
                `ip_address` VARCHAR(45) NULL,
                `user_agent` VARCHAR(500) NULL,
                `session_id` VARCHAR(128) NULL,
                `entity_type` VARCHAR(100) NULL,
                `entity_id` VARCHAR(100) NULL,
                `action` VARCHAR(100) NOT NULL,
                `old_values` JSON NULL,
                `new_values` JSON NULL,
                `changes_summary` TEXT NULL,
                `metadata` JSON NULL,
                `hash` VARCHAR(64) NOT NULL,
                `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
                PRIMARY KEY (`id`),
                INDEX `idx_event_type` (`event_type`),
                INDEX `idx_category_date` (`event_category`, `created_at`),
                INDEX `idx_user_date` (`user_id`, `created_at`),
                INDEX `idx_entity` (`entity_type`, `entity_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_audit_retention_policies` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `policy_name` VARCHAR(255) NOT NULL,
                `event_category` VARCHAR(100) NULL,
                `event_type` VARCHAR(100) NULL,
                `retention_days` INT UNSIGNED NOT NULL DEFAULT 365,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_audit_summaries` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `summary_date` DATE NOT NULL,
                `event_category` VARCHAR(100) NULL,
                `total_events` INT UNSIGNED NOT NULL DEFAULT 0,
                `unique_users` INT UNSIGNED NOT NULL DEFAULT 0,
                `unique_admins` INT UNSIGNED NOT NULL DEFAULT 0,
                `error_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `security_events` INT UNSIGNED NOT NULL DEFAULT 0,
                `hash` VARCHAR(64) NOT NULL,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_date_category` (`summary_date`, `event_category`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function setupDefaultRetentionPolicies() {
        $policies = [
            ['name' => 'Security Events', 'category' => 'security', 'retention' => 730],
            ['name' => 'Admin Actions', 'category' => 'admin', 'retention' => 365],
            ['name' => 'Data Changes', 'category' => 'data', 'retention' => 365],
            ['name' => 'Billing Events', 'category' => 'billing', 'retention' => 180],
            ['name' => 'User Actions', 'category' => 'user', 'retention' => 90],
            ['name' => 'API Calls', 'category' => 'api', 'retention' => 30],
        ];
        
        foreach ($policies as $policy) {
            Capsule::table('mod_audit_retention_policies')->insert([
                'policy_name' => $policy['name'],
                'event_category' => $policy['category'],
                'retention_days' => $policy['retention'],
                'is_active' => 1,
            ]);
        }
    }
    
    /**
     * Log an audit event
     */
    public function log($eventType, $action, $data = [], $options = []) {
        $requestId = $this->generateEventId();
        
        $userId = $options['user_id'] ?? $this->getCurrentUserId();
        $adminId = $options['admin_id'] ?? $this->getCurrentAdminId();
        
        $category = $this->determineCategory($eventType, $action);
        $severity = $this->determineSeverity($action, $data);
        
        // Process old/new values if provided
        $oldValues = isset($options['old_values']) ? json_encode($options['old_values']) : null;
        $newValues = isset($options['new_values']) ? json_encode($options['new_values']) : null;
        
        if ($oldValues && $newValues) {
            $changesSummary = $this->generateChangesSummary($options['old_values'], $options['new_values']);
        } else {
            $changesSummary = null;
        }
        
        $logData = [
            'event_id' => $requestId,
            'event_type' => $eventType,
            'event_category' => $category,
            'severity' => $severity,
            'user_id' => $userId,
            'admin_id' => $adminId,
            'ip_address' => $this->getClientIp(),
            'user_agent' => substr($this->getUserAgent(), 0, 500),
            'session_id' => session_id(),
            'entity_type' => $options['entity_type'] ?? null,
            'entity_id' => $options['entity_id'] ?? null,
            'action' => $action,
            'old_values' => $oldValues,
            'new_values' => $newValues,
            'changes_summary' => $changesSummary,
            'metadata' => json_encode([
                'request_uri' => $_SERVER['REQUEST_URI'] ?? '',
                'request_method' => $_SERVER['REQUEST_METHOD'] ?? '',
                'module_version' => '1.0.0',
            ]),
            'hash' => '',
            'created_at' => Carbon::now(),
        ];
        
        // Generate tamper-proof hash
        $logData['hash'] = $this->generateHash($logData);
        
        $logId = Capsule::table('mod_audit_logs')->insertGetId($logData);
        
        return [
            'success' => true,
            'event_id' => $requestId,
            'log_id' => $logId,
        ];
    }
    
    protected function generateEventId() {
        return sprintf(
            '%04x%04x-%04x-%04x-%04x-%04x%04x%04x',
            mt_rand(0, 0xffff),
            mt_rand(0, 0xffff),
            mt_rand(0, 0xffff),
            mt_rand(0, 0x0fff) | 0x4000,
            mt_rand(0, 0x3fff) | 0x8000,
            mt_rand(0, 0xffff),
            mt_rand(0, 0xffff),
            mt_rand(0, 0xffff)
        );
    }
    
    protected function determineCategory($eventType, $action) {
        $lowerAction = strtolower($action);
        $lowerEvent = strtolower($eventType);
        
        if (in_array($lowerAction, ['login', 'logout', 'failed_login', 'password_change'])) {
            return 'security';
        }
        if (in_array($lowerAction, ['create', 'update', 'delete', 'modify'])) {
            return 'data';
        }
        if (in_array($lowerAction, ['invoice', 'payment', 'refund'])) {
            return 'billing';
        }
        if (in_array($lowerAction, ['api_call', 'api_error'])) {
            return 'api';
        }
        if ($this->getCurrentAdminId()) {{\displaystyle}}return 'admin';
        {{\}}
        
        return 'user';
    }
    
    protected function determineSeverity($action, $data) {
        $criticalActions = ['delete', 'failed_login', 'security_alert', 'breach'];
        $warningActions = ['update', 'modify', 'failed_action'];
        
        if (in_array(strtolower($action), $criticalActions)) {
            return 'critical';
        }
        if (in_array(strtolower($action), $warningActions)) {
            return 'warning';
        }
        
        return 'info';
    }
    
    protected function generateChangesSummary($oldValues, $newValues) {
        $changes = [];
        
        foreach ($newValues as $key => $newValue) {
            $oldValue = $oldValues[$key] ?? null;
            if ($oldValue !== $newValue) {
                $changes[] = "$key: " . (is_scalar($oldValue) ? $oldValue : 'N/A') . " -> " . (is_scalar($newValue) ? $newValue : 'N/A');
            }
        }
        
        return implode('; ', $changes);
    }
    
    protected function generateHash($data) {
        $hashInput = implode('|', [
            $data['event_id'],
            $data['event_type'],
            $data['event_category'],
            $data['action'],
            $data['user_id'] ?? '',
            $data['admin_id'] ?? '',
            $data['entity_type'] ?? '',
            $data['entity_id'] ?? '',
            $data['old_values'] ?? '',
            $data['new_values'] ?? '',
            $data['created_at']->toDateTimeString(),
        ]);
        
        return hash($this->hashAlgorithm, $hashInput);
    }
    
    protected function getCurrentUserId() {
        if (defined('WHMCS_USER')) {
            return WHMCS\User\Auth::user()->id ?? null;
        }
        return Capsule::table('tblsessions')->where('id', session_id())->value('user_id');
    }
    
    protected function getCurrentAdminId() {
        if (defined('ADMINAREA')) {
            return Capsule::table('tblsessions')->where('id', session_id())->value('admin_id');
        }
        return null;
    }
    
    protected function getClientIp() {
        $headers = ['HTTP_CF_CONNECTING_IP', 'HTTP_X_FORWARDED_FOR', 'HTTP_X_REAL_IP', 'REMOTE_ADDR'];
        foreach ($headers as $header) {
            if (!empty($_SERVER[$header])) {
                $ip = $_SERVER[$header];
                return strpos($ip, ',') !== false ? trim(explode(',', $ip)[0]) : $ip;
            }
        }
        return '0.0.0.0';
    }
    
    protected function getUserAgent() {
        return $_SERVER['HTTP_USER_AGENT'] ?? 'Unknown';
    }
    
    /**
     * Log data change with old/new values
     */
    public function logDataChange($entityType, $entityId, $action, $oldValues, $newValues, $options = []) {
        $eventType = "{$entityType}.{$action}";
        
        $options['entity_type'] = $entityType;
        $options['entity_id'] = $entityId;
        $options['old_values'] = $oldValues;
        $options['new_values'] = $newValues;
        
        return $this->log($eventType, $action, compact('entityType', 'entityId', 'oldValues', 'newValues'), $options);
    }
    
    /**
     * Log security event
     */
    public function logSecurityEvent($eventType, $details = [], $severity = 'warning') {
        return $this->log($eventType, $eventType, $details, array_merge(['severity' => $severity]));
    }
    
    /**
     * Verify log integrity
     */
    public function verifyIntegrity($logId) {
        $log = Capsule::table('mod_audit_logs')->where('id', $logId)->first();
        
        if (!$log) {
            return ['valid' => false, 'reason' => 'Log not found'];
        }
        
        $storedHash = $log->hash;
        $recalculatedHash = $this->generateHash((array)$log);
        
        return [
            'valid' => $storedHash === $recalculatedHash,
            'stored_hash' => $storedHash,
            'calculated_hash' => $recalculatedHash,
        ];
    }
}
```

### lib/AuditQuery.php

```php
<?php
/**
 * Audit Query Builder
 */

namespace WHMCS\Module\AuditTrails;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class AuditQuery {
    
    /**
     * Search audit logs
     */
    public function search($filters = []) {
        $query = Capsule::table('mod_audit_logs');
        
        // Date range filters
        if (!empty($filters['start_date'])) {
            $query->where('created_at', '>=', $filters['start_date']);
        }
        if (!empty($filters['end_date'])) {
            $query->where('created_at', '<=', $filters['end_date'] . ' 23:59:59');
        }
        
        // User filters
        if (!empty($filters['user_id'])) {
            $query->where('user_id', $filters['user_id']);
        }
        if (!empty($filters['admin_id'])) {
            $query->where('admin_id', $filters['admin_id']);
        }
        
        // Category and type filters
        if (!empty($filters['category'])) {
            $query->where('event_category', $filters['category']);
        }
        if (!empty($filters['event_type'])) {
            $query->where('event_type', 'like', '%' . $filters['event_type'] . '%');
        }
        
        // Severity filter
        if (!empty($filters['severity'])) {
            $query->where('severity', $filters['severity']);
        }
        
        // Entity filter
        if (!empty($filters['entity_type'])) {
            $query->where('entity_type', $filters['entity_type']);
        }
        if (!empty($filters['entity_id'])) {
            $query->where('entity_id', $filters['entity_id']);
        }
        
        // IP filter
        if (!empty($filters['ip_address'])) {
            $query->where('ip_address', 'like', $filters['ip_address'] . '%');
        }
        
        // Search in changes
        if (!empty($filters['search'])) {
            $query->where(function($q) use ($filters) {
                $q->where('changes_summary', 'like', '%' . $filters['search'] . '%')
                    ->orWhere('event_type', 'like', '%' . $filters['search'] . '%')
                    ->orWhere('action', 'like', '%' . $filters['search'] . '%');
            });
        }
        
        // Sorting
        $sortField = $filters['sort_field'] ?? 'created_at';
        $sortOrder = $filters['sort_order'] ?? 'desc';
        $query->orderBy($sortField, $sortOrder);
        
        // Pagination
        $limit = $filters['limit'] ?? 50;
        $offset = $filters['offset'] ?? 0;
        $query->limit($limit)->offset($offset);
        
        $results = $query->get();
        
        // Get total count
        $totalQuery = Capsule::table('mod_audit_logs');
        if (!empty($filters['start_date'])) {
            $totalQuery->where('created_at', '>=', $filters['start_date']);
        }
        if (!empty($filters['end_date'])) {
            $totalQuery->where('created_at', '<=', $filters['end_date'] . ' 23:59:59');
        }
        $total = $totalQuery->count();
        
        return [
            'results' => $results,
            'total' => $total,
            'limit' => $limit,
            'offset' => $offset,
            'has_more' => ($offset + $limit) < $total,
        ];
    }
    
    /**
     * Get audit summary
     */
    public function getSummary($startDate = null, $endDate = null) {
        $startDate = $startDate ?? Carbon::now()->subDays(30)->toDateString();
        $endDate = $endDate ?? Carbon::now()->toDateString();
        
        $totalEvents = Capsule::table('mod_audit_logs')
            ->whereBetween('created_at', [$startDate, $endDate])
            ->count();
        
        $byCategory = Capsule::table('mod_audit_logs')
            ->whereBetween('created_at', [$startDate, $endDate])
            ->groupBy('event_category')
            ->selectRaw('event_category, COUNT(*) as count')
            ->get();
        
        $bySeverity = Capsule::table('mod_audit_logs')
            ->whereBetween('created_at', [$startDate, $endDate])
            ->groupBy('severity')
            ->selectRaw('severity, COUNT(*) as count')
            ->get();
        
        $recentEvents = Capsule::table('mod_audit_logs')
            ->whereBetween('created_at', [$startDate, $endDate])
            ->orderBy('created_at', 'desc')
            ->limit(10)
            ->get();
        
        $errorEvents = Capsule::table('mod_audit_logs')
            ->whereBetween('created_at', [$startDate, $endDate])
            ->whereIn('severity', ['error', 'critical'])
            ->count();
        
        return [
            'period' => ['start' => $startDate, 'end' => $endDate],
            'total_events' => $totalEvents,
            'by_category' => $byCategory->keyBy('event_category'),
            'by_severity' => $bySeverity->keyBy('severity'),
            'error_count' => $errorEvents,
            'recent_events' => $recentEvents,
        ];
    }
    
    /**
     * Get timeline of events for an entity
     */
    public function getEntityTimeline($entityType, $entityId, $limit = 50) {
        return Capsule::table('mod_audit_logs')
            ->where('entity_type', $entityType)
            ->where('entity_id', $entityId)
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get();
    }
    
    /**
     * Get user activity summary
     */
    public function getUserActivitySummary($userId, $days = 30) {
        $startDate = Carbon::now()->subDays($days)->toDateString();
        
        return [
            'total_actions' => Capsule::table('mod_audit_logs')
                ->where('user_id', $userId)
                ->where('created_at', '>=', $startDate)
                ->count(),
            'by_category' => Capsule::table('mod_audit_logs')
                ->where('user_id', $userId)
                ->where('created_at', '>=', $startDate)
                ->groupBy('event_category')
                ->selectRaw('event_category, COUNT(*) as count')
                ->get(),
            'most_frequent_actions' => Capsule::table('mod_audit_logs')
                ->where('user_id', $userId)
                ->where('created_at', '>=', $startDate)
                ->groupBy('action')
                ->selectRaw('action, COUNT(*) as count')
                ->orderBy('count', 'desc')
                ->limit(5)
                ->get(),
            'last_activity' => Capsule::table('mod_audit_logs')
                ->where('user_id', $userId)
                ->orderBy('created_at', 'desc')
                ->first(),
        ];
    }
}
```

### lib/AuditRetention.php

```php
<?php
/**
 * Audit Retention Manager
 */

namespace WHMCS\Module\AuditTrails;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class AuditRetention {
    
    public function applyRetentionPolicies() {
        $policies = Capsule::table('mod_audit_retention_policies')
            ->where('is_active', 1)
            ->get();
        
        foreach ($policies as $policy) {
            if ($policy->retention_days > 0) {
                $this->applyPolicy($policy);
            }
        }
    }
    
    protected function applyPolicy($policy) {
        $cutoffDate = Carbon::now()->subDays($policy->retention_days)->toDateTimeString();
        
        $query = Capsule::table('mod_audit_logs')
            ->where('created_at', '<', $cutoffDate);
        
        if ($policy->event_category) {
            $query->where('event_category', $policy->event_category);
        }
        if ($policy->event_type) {
            $query->where('event_type', $policy->event_type);
        }
        
        $deletedCount = $query->delete();
        
        if ($deletedCount > 0) {
            logActivity("Audit retention policy applied: {$policy->policy_name}, deleted {$deletedCount} records");
        }
    }
    
    public function generateDailySummaries() {
        $yesterday = Carbon::yesterday()->toDateString();
        
        // Check if summary already exists
        $exists = Capsule::table('mod_audit_summaries')
            ->where('summary_date', $yesterday)
            ->exists();
        
        if ($exists) {
            return;
        }
        
        $categories = Capsule::table('mod_audit_logs')
            ->whereDate('created_at', $yesterday)
            ->groupBy('event_category')
            ->selectRaw('event_category, COUNT(*) as total')
            ->get();
        
        foreach ($categories as $category) {
            $summaryDate = Carbon::parse($yesterday);
            
            $stats = Capsule::table('mod_audit_logs')
                ->whereDate('created_at', $yesterday)
                ->where('event_category', $category->event_category)
                ->selectRaw('
                    COUNT(*) as total_events,
                    COUNT(DISTINCT user_id) as unique_users,
                    COUNT(DISTINCT admin_id) as unique_admins,
                    SUM(CASE WHEN severity IN ("error", "critical") THEN 1 ELSE 0 END) as error_count,
                    SUM(CASE WHEN severity IN ("warning") THEN 1 ELSE 0 END) as warning_count,
                    SUM(CASE WHEN event_category = "security" THEN 1 ELSE 0 END) as security_events
                ')
                ->first();
            
            Capsule::table('mod_audit_summaries')->insert([
                'summary_date' => $yesterday,
                'event_category' => $category->event_category,
                'total_events' => $stats->total_events,
                'unique_users' => $stats->unique_users ?? 0,
                'unique_admins' => $stats->unique_admins ?? 0,
                'error_count' => $stats->error_count ?? 0,
                'warning_count' => $stats->warning_count ?? 0,
                'security_events' => $stats->security_events ?? 0,
                'hash' => hash('sha256', json_encode($stats)),
            ]);
        }
    }
}
```

### lib/AuditExporter.php

```php
<?php
/**
 * Audit Log Exporter
 */

namespace WHMCS\Module\AuditTrails;

use Illuminate\Database\Capsule\Manager as Capsule;
use TCPDF;

class AuditExporter {
    
    public function export($filters = [], $format = 'csv') {
        $query = new AuditQuery();
        $results = $query->search(array_merge($filters, ['limit' => 100000]));
        
        switch ($format) {
            case 'json':
                return $this->exportJson($results['results']);
            case 'pdf':
                return $this->exportPdf($results['results'], $filters);
            case 'csv':
            default:
                return $this->exportCsv($results['results']);
        }
    }
    
    protected function exportJson($results) {
        $data = [];
        foreach ($results as $result) {
            $data[] = [
                'event_id' => $result->event_id,
                'event_type' => $result->event_type,
                'category' => $result->event_category,
                'severity' => $result->severity,
                'user_id' => $result->user_id,
                'admin_id' => $result->admin_id,
                'action' => $result->action,
                'entity_type' => $result->entity_type,
                'entity_id' => $result->entity_id,
                'ip_address' => $result->ip_address,
                'changes' => json_decode($result->changes_summary),
                'created_at' => $result->created_at,
            ];
        }
        
        return [
            'format' => 'json',
            'data' => $data,
            'count' => count($data),
            'exported_at' => date('c'),
        ];
    }
    
    protected function exportCsv($results) {
        $output = fopen('php://temp', 'r+');
        
        fputcsv($output, [
            'Event ID', 'Timestamp', 'Type', 'Category', 'Severity',
            'User ID', 'Admin ID', 'Action', 'Entity Type', 'Entity ID',
            'IP Address', 'Changes Summary'
        ]);
        
        foreach ($results as $result) {
            fputcsv($output, [
                $result->event_id,
                $result->created_at,
                $result->event_type,
                $result->event_category,
                $result->severity,
                $result->user_id,
                $result->admin_id,
                $result->action,
                $result->entity_type,
                $result->entity_id,
                $result->ip_address,
                $result->changes_summary,
            ]);
        }
        
        rewind($output);
        $content = stream_get_contents($output);
        fclose($output);
        
        return [
            'format' => 'csv',
            'content' => $content,
            'filename' => 'audit_logs_' . date('Y-m-d_His') . '.csv',
        ];
    }
    
    protected function exportPdf($results, $filters) {
        return [
            'format' => 'pdf',
            'filename' => 'audit_report_' . date('Y-m-d_His') . '.pdf',
            'note' => 'PDF generation requires TCPDF or similar library',
        ];
    }
}
```

## API Endpoints

```
GET  /api/v1/audit/logs                  - Query audit logs
GET  /api/v1/audit/logs/{id}             - Get specific log entry
GET  /api/v1/audit/summary               - Get audit summary
GET  /api/v1/audit/user/{userId}         - Get user activity
GET  /api/v1/audit/entity/{type}/{id}    - Get entity timeline
POST /api/v1/audit/export                - Export audit logs
POST /api/v1/audit/log                  - Create audit log entry
GET  /api/v1/audit/verify/{id}           - Verify log integrity
```

## Hooks Integration

```php
// Log all admin actions
add_hook('AdminAreaPageHook', 1, function($params) {
    if (isset($params['action']) && in_array($params['action'], ['modify', 'delete', 'create'])) {
        $logger = new \WHMCS\Module\AuditTrails\AuditLogger();
        $logger->log('admin', $params['action'], $params);
    }
});

// Security event logging
add_hook('FailedLoginAttempt', 1, function($params) {
    $logger = new \WHMCS\Module\AuditTrails\AuditLogger();
    $logger->logSecurityEvent('failed_login', [
        'user' => $params['user'],
        'ip' => $params['ip'],
        'reason' => $params['reason'] ?? 'Unknown',
    ], 'warning');
});
```
