# WHMCS Audit Logging Skill

## Purpose
Provides patterns for implementing comprehensive audit logging in WHMCS, tracking all system activities, user actions, data changes, and security events for compliance and troubleshooting.

## Implementation Patterns

### Audit Logger
```php
<?php
// includes/AuditLogging.class.php

class AuditLogger {
    private $db;
    private $config;
    private $buffer = [];
    private $bufferSize = 100;
    
    public function __construct() {
        $this->db = console::db();
        $this->loadConfig();
    }
    
    private function loadConfig() {
        $this->config = [
            'log_level' => getenv('AUDIT_LOG_LEVEL') ?: 'info',
            'buffer_size' => (int)(getenv('AUDIT_BUFFER_SIZE') ?: 100),
            'async_enabled' => getenv('AUDIT_ASYNC') ?: true,
            'retention_days' => (int)(getenv('AUDIT_RETENTION_DAYS') ?: 365),
            'sensitive_fields' => ['password', 'credit_card', 'ssn', 'api_key', 'secret']
        ];
    }
    
    // Log an audit event
    public function log($category, $action, $data = [], $context = []) {
        $event = $this->prepareEvent($category, $action, $data, $context);
        
        if ($this->shouldLog($event)) {
            $this->buffer[] = $event;
            
            if (count($this->buffer) >= $this->bufferSize) {
                $this->flush();
            }
        }
        
        return $event['id'];
    }
    
    private function prepareEvent($category, $action, $data, $context) {
        return [
            'id' => $this->generateEventId(),
            'timestamp' => date('Y-m-d H:i:s'),
            'category' => $category,
            'action' => $action,
            'actor_type' => $context['actor_type'] ?? $this->getActorType(),
            'actor_id' => $context['actor_id'] ?? $this->getActorId(),
            'actor_ip' => $context['ip'] ?? $this->getClientIP(),
            'actor_user_agent' => $context['user_agent'] ?? $this->getUserAgent(),
            'target_type' => $context['target_type'] ?? null,
            'target_id' => $context['target_id'] ?? null,
            'data' => $this->sanitizeData($data),
            'result' => $context['result'] ?? 'success',
            'error_message' => $context['error_message'] ?? null,
            'duration_ms' => $context['duration_ms'] ?? null,
            'session_id' => $context['session_id'] ?? $this->getSessionId(),
            'request_id' => $context['request_id'] ?? $this->getRequestId()
        ];
    }
    
    private function sanitizeData($data) {
        if (!is_array($data)) {
            return $data;
        }
        
        $sanitized = [];
        foreach ($data as $key => $value) {
            $isSensitive = false;
            foreach ($this->config['sensitive_fields'] as $field) {
                if (stripos($key, $field) !== false) {
                    $isSensitive = true;
                    break;
                }
            }
            
            if ($isSensitive) {
                $sanitized[$key] = '[REDACTED]';
            } elseif (is_array($value)) {
                $sanitized[$key] = $this->sanitizeData($value);
            } else {
                $sanitized[$key] = $value;
            }
        }
        
        return $sanitized;
    }
    
    private function shouldLog($event) {
        $logLevels = ['debug' => 0, 'info' => 1, 'warning' => 2, 'error' => 3, 'critical' => 4];
        $currentLevel = $logLevels[$this->config['log_level']];
        
        // Always log security events
        if (in_array($event['category'], ['security', 'compliance', 'admin'])) {
            return true;
        }
        
        return $currentLevel <= $this->determineEventLevel($event);
    }
    
    private function determineEventLevel($event) {
        if ($event['result'] === 'error') return 3;
        if ($event['category'] === 'security') return 4;
        if ($event['action'] === 'delete') return 3;
        return 1;
    }
    
    // Flush buffer to database
    public function flush() {
        if (empty($this->buffer)) {
            return;
        }
        
        $this->db->insert('mod_audit_log', $this->buffer);
        $this->buffer = [];
    }
    
    // Automatic flush on shutdown
    public function __destruct() {
        $this->flush();
    }
}
```

### Audit Event Categories
```php
// Predefined category constants
class AuditCategory {
    const SECURITY = 'security';
    const ADMIN = 'admin';
    const USER = 'user';
    const DATA = 'data';
    const SYSTEM = 'system';
    const API = 'api';
    const PAYMENT = 'payment';
    const SERVICE = 'service';
    const COMPLIANCE = 'compliance';
    const AUTHENTICATION = 'authentication';
}

// Security events
class SecurityAuditLogger {
    public function logLoginAttempt($userId, $result, $metadata = []) {
        return $this->log('security', 'login_attempt', [
            'user_id' => $userId,
            'result' => $result
        ], [
            'target_type' => 'user',
            'target_id' => $userId,
            'result' => $result,
            'duration_ms' => $metadata['duration_ms'] ?? null
        ]);
    }
    
    public function logFailedLogin($userId, $reason, $ip) {
        return $this->log('security', 'failed_login', [
            'user_id' => $userId,
            'reason' => $reason,
            'attempt_count' => $this->getFailedAttemptCount($userId)
        ], [
            'ip' => $ip,
            'result' => 'failure',
            'error_message' => $reason
        ]);
    }
    
    public function logPermissionChange($adminId, $targetUserId, $changes) {
        return $this->log('admin', 'permission_change', [
            'admin_id' => $adminId,
            'target_user_id' => $targetUserId,
            'changes' => $changes
        ], [
            'actor_type' => 'admin',
            'actor_id' => $adminId,
            'target_type' => 'user',
            'target_id' => $targetUserId
        ]);
    }
    
    public function logAccessDenied($userId, $resource, $reason) {
        return $this->log('security', 'access_denied', [
            'user_id' => $userId,
            'resource' => $resource,
            'reason' => $reason
        ], [
            'target_type' => 'resource',
            'target_id' => $resource,
            'result' => 'denied',
            'error_message' => $reason
        ]);
    }
    
    public function logDataExport($userId, $dataType, $recordCount) {
        return $this->log('data', 'export', [
            'user_id' => $userId,
            'data_type' => $dataType,
            'record_count' => $recordCount
        ], [
            'target_type' => $dataType,
            'result' => 'success'
        ]);
    }
}

// Payment audit logging
class PaymentAuditLogger {
    public function logPayment($clientId, $invoiceId, $amount, $method, $result) {
        return $this->log('payment', 'payment_processed', [
            'client_id' => $clientId,
            'invoice_id' => $invoiceId,
            'amount' => $amount,
            'method' => $method,
            'result' => $result
        ], [
            'target_type' => 'invoice',
            'target_id' => $invoiceId,
            'result' => $result
        ]);
    }
    
    public function logRefund($clientId, $invoiceId, $amount, $reason) {
        return $this->log('payment', 'refund_issued', [
            'client_id' => $clientId,
            'invoice_id' => $invoiceId,
            'amount' => $amount,
            'reason' => $reason
        ], [
            'target_type' => 'invoice',
            'target_id' => $invoiceId
        ]);
    }
    
    public function logChargeback($clientId, $invoiceId, $amount) {
        return $this->log('payment', 'chargeback', [
            'client_id' => $clientId,
            'invoice_id' => $invoiceId,
            'amount' => $amount
        ], [
            'target_type' => 'invoice',
            'target_id' => $invoiceId,
            'result' => 'dispute'
        ]);
    }
}
```

### Audit Query Builder
```php
class AuditQueryBuilder {
    private $db;
    
    public function __construct() {
        $this->db = console::db();
    }
    
    // Query audit logs with filters
    public function query($filters = []) {
        $defaults = [
            'category' => null,
            'action' => null,
            'actor_id' => null,
            'target_type' => null,
            'target_id' => null,
            'start_date' => null,
            'end_date' => null,
            'result' => null,
            'ip' => null,
            'limit' => 100,
            'offset' => 0,
            'order' => 'desc'
        ];
        
        $filters = array_merge($defaults, $filters);
        
        $query = "SELECT * FROM mod_audit_log WHERE 1=1";
        $params = [];
        
        if ($filters['category']) {
            $query .= " AND category = ?";
            $params[] = $filters['category'];
        }
        
        if ($filters['action']) {
            $query .= " AND action LIKE ?";
            $params[] = '%' . $filters['action'] . '%';
        }
        
        if ($filters['actor_id']) {
            $query .= " AND actor_id = ?";
            $params[] = $filters['actor_id'];
        }
        
        if ($filters['target_type']) {
            $query .= " AND target_type = ?";
            $params[] = $filters['target_type'];
        }
        
        if ($filters['target_id']) {
            $query .= " AND target_id = ?";
            $params[] = $filters['target_id'];
        }
        
        if ($filters['start_date']) {
            $query .= " AND timestamp >= ?";
            $params[] = $filters['start_date'];
        }
        
        if ($filters['end_date']) {
            $query .= " AND timestamp <= ?";
            $params[] = $filters['end_date'];
        }
        
        if ($filters['result']) {
            $query .= " AND result = ?";
            $params[] = $filters['result'];
        }
        
        if ($filters['ip']) {
            $query .= " AND actor_ip LIKE ?";
            $params[] = $filters['ip'] . '%';
        }
        
        $query .= " ORDER BY timestamp " . strtoupper($filters['order']);
        $query .= " LIMIT {$filters['limit']} OFFSET {$filters['offset']}";
        
        return $this->db->select($query, $params);
    }
    
    // Get audit trail for a specific entity
    public function getEntityTrail($entityType, $entityId, $limit = 50) {
        return $this->query([
            'target_type' => $entityType,
            'target_id' => $entityId,
            'limit' => $limit
        ]);
    }
    
    // Get user activity summary
    public function getUserActivity($userId, $period = '30d') {
        return $this->db->select(
            "SELECT 
                DATE(timestamp) as date,
                COUNT(*) as action_count,
                COUNT(DISTINCT category) as categories,
                COUNT(CASE WHEN result = 'error' THEN 1 END) as errors
             FROM mod_audit_log
             WHERE actor_id = ? AND timestamp > DATE_SUB(NOW(), INTERVAL ?)
             GROUP BY DATE(timestamp)
             ORDER BY date DESC",
            [$userId, $period]
        );
    }
    
    // Get security events summary
    public function getSecuritySummary($period = '24h') {
        return $this->db->select(
            "SELECT 
                action,
                COUNT(*) as count,
                COUNT(DISTINCT actor_ip) as unique_ips,
                MAX(timestamp) as last_occurrence
             FROM mod_audit_log
             WHERE category = 'security' AND timestamp > DATE_SUB(NOW(), INTERVAL ?)
             GROUP BY action
             ORDER BY count DESC",
            [$period]
        );
    }
    
    // Search audit logs
    public function search($keyword, $filters = []) {
        $query = "SELECT * FROM mod_audit_log WHERE 1=1";
        $params = [];
        
        if ($keyword) {
            $query .= " AND (action LIKE ? OR data LIKE ? OR error_message LIKE ?)";
            $searchTerm = '%' . $keyword . '%';
            $params[] = $searchTerm;
            $params[] = $searchTerm;
            $params[] = $searchTerm;
        }
        
        // Apply additional filters
        $baseQuery = $this->query($filters);
        
        // Merge conditions
        if (!empty($baseQuery)) {
            // Filter the results
            return array_filter($baseQuery, function($log) use ($keyword) {
                return stripos($log['action'], $keyword) !== false ||
                       stripos(json_encode($log['data']), $keyword) !== false;
            });
        }
        
        return [];
    }
}
```

### Audit Report Generator
```php
class AuditReportGenerator {
    public function generateReport($config = []) {
        $defaults = [
            'start_date' => date('Y-m-d', strtotime('-30 days')),
            'end_date' => date('Y-m-d'),
            'categories' => null,
            'include_details' => true
        ];
        
        $config = array_merge($defaults, $config);
        
        return [
            'report_info' => $this->getReportInfo($config),
            'summary' => $this->getSummaryStats($config),
            'activity_by_category' => $this->getActivityByCategory($config),
            'activity_by_user' => $this->getActivityByUser($config),
            'security_events' => $this->getSecurityEvents($config),
            'failed_actions' => $this->getFailedActions($config),
            'top_targets' => $this->getTopTargets($config),
            'geo_distribution' => $this->getGeoDistribution($config),
            'timeline' => $this->getTimeline($config)
        ];
    }
    
    private function getSummaryStats($config) {
        $stats = $this->db->select(
            "SELECT 
                COUNT(*) as total_events,
                COUNT(DISTINCT actor_id) as unique_actors,
                COUNT(DISTINCT actor_ip) as unique_ips,
                COUNT(CASE WHEN category = 'security' THEN 1 END) as security_events,
                COUNT(CASE WHEN result = 'error' THEN 1 END) as errors
             FROM mod_audit_log
             WHERE timestamp BETWEEN ? AND ?",
            [$config['start_date'], $config['end_date'] . ' 23:59:59']
        );
        
        return $stats;
    }
    
    private function getActivityByCategory($config) {
        return $this->db->select(
            "SELECT 
                category,
                action,
                COUNT(*) as count
             FROM mod_audit_log
             WHERE timestamp BETWEEN ? AND ?
             GROUP BY category, action
             ORDER BY count DESC",
            [$config['start_date'], $config['end_date'] . ' 23:59:59']
        );
    }
    
    private function getActivityByUser($config) {
        return $this->db->select(
            "SELECT 
                actor_id,
                actor_type,
                COUNT(*) as action_count,
                COUNT(DISTINCT action) as unique_actions,
                COUNT(CASE WHEN result = 'error' THEN 1 END) as errors
             FROM mod_audit_log
             WHERE timestamp BETWEEN ? AND ?
             GROUP BY actor_id, actor_type
             ORDER BY action_count DESC
             LIMIT 50",
            [$config['start_date'], $config['end_date'] . ' 23:59:59']
        );
    }
    
    public function exportCSV($report, $filename = null) {
        $filename = $filename ?: 'audit_report_' . date('Y-m-d') . '.csv';
        
        $handle = fopen('php://temp', 'r+');
        
        // Header row
        fputcsv($handle, [
            'Timestamp', 'Category', 'Action', 'Actor ID', 'Actor Type',
            'Target Type', 'Target ID', 'Result', 'IP Address'
        ]);
        
        // Data rows
        foreach ($report['details'] as $row) {
            fputcsv($handle, [
                $row['timestamp'],
                $row['category'],
                $row['action'],
                $row['actor_id'],
                $row['actor_type'],
                $row['target_type'],
                $row['target_id'],
                $row['result'],
                $row['actor_ip']
            ]);
        }
        
        rewind($handle);
        $content = stream_get_contents($handle);
        fclose($handle);
        
        return $content;
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_audit_log (
    id VARCHAR(64) PRIMARY KEY,
    timestamp DATETIME NOT NULL,
    category VARCHAR(50) NOT NULL,
    action VARCHAR(100) NOT NULL,
    actor_type VARCHAR(20),
    actor_id VARCHAR(64),
    actor_ip VARCHAR(45),
    actor_user_agent VARCHAR(500),
    target_type VARCHAR(50),
    target_id VARCHAR(64),
    data JSON,
    result ENUM('success', 'failure', 'denied', 'error') DEFAULT 'success',
    error_message TEXT,
    duration_ms INT,
    session_id VARCHAR(64),
    request_id VARCHAR(64),
    INDEX idx_timestamp (timestamp),
    INDEX idx_category (category),
    INDEX idx_actor (actor_type, actor_id),
    INDEX idx_target (target_type, target_id),
    INDEX idx_result (result),
    INDEX idx_action (action)
);

CREATE TABLE mod_audit_log_archive (
    id VARCHAR(64) PRIMARY KEY,
    timestamp DATETIME NOT NULL,
    category VARCHAR(50) NOT NULL,
    action VARCHAR(100) NOT NULL,
    actor_type VARCHAR(20),
    actor_id VARCHAR(64),
    actor_ip VARCHAR(45),
    target_type VARCHAR(50),
    target_id VARCHAR(64),
    data JSON,
    result VARCHAR(20),
    archived_at DATETIME
);

CREATE TABLE mod_audit_saved_filters (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    user_id INT,
    filters JSON,
    is_shared TINYINT(1) DEFAULT 0,
    created_at DATETIME,
    INDEX idx_user (user_id)
);

CREATE TABLE mod_audit_reports (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    report_type VARCHAR(50),
    config JSON,
    generated_by INT,
    generated_at DATETIME,
    file_path VARCHAR(500),
    INDEX idx_generated (generated_at)
);
```

## Usage Examples

### Log a Custom Event
```php
$logger = new AuditLogger();
$logger->log('admin', 'configuration_change', [
    'setting' => 'max_upload_size',
    'old_value' => 10485760,
    'new_value' => 20971520
], [
    'actor_id' => $adminId,
    'target_type' => 'config',
    'target_id' => 'upload_settings'
]);
```

### Query Security Events
```php
$query = new AuditQueryBuilder();
$events = $query->query([
    'category' => 'security',
    'start_date' => date('Y-m-d', strtotime('-7 days')),
    'limit' => 50
]);

foreach ($events as $event) {
    echo "[{$event['timestamp']}] {$event['action']} - {$event['result']}\n";
}
```

### Generate Audit Report
```php
$generator = new AuditReportGenerator();
$report = $generator->generateReport([
    'start_date' => '2026-01-01',
    'end_date' => '2026-03-31'
]);

echo "Total Events: {$report['summary']['total_events']}\n";
echo "Security Events: {$report['summary']['security_events']}\n";
```

### Get User Activity Trail
```php
$query = new AuditQueryBuilder();
$trail = $query->getEntityTrail('user', $userId);

foreach ($trail as $event) {
    echo "{$event['timestamp']}: {$event['action']} on {$event['target_type']}:{$event['target_id']}\n";
}
```

## Best Practices

1. **Log everything important**: Actions that affect data, security, or compliance
2. **Use consistent categories**: Standardize naming for easier querying
3. **Sanitize sensitive data**: Never log passwords, credit cards, or secrets
4. **Buffer writes**: Use memory buffering for high-volume logging
5. **Archive old logs**: Move historical logs to cheaper storage
6. **Implement retention policies**: Delete old logs based on requirements
7. **Correlate events**: Use request_id and session_id to trace related events
8. **Monitor log health**: Track log volume, errors, and storage usage