# WHMCS Audit Trail Implementation Workflow

## Purpose

Implement comprehensive audit trail capabilities for WHMCS to track user actions, system changes, and data modifications for security, compliance, and troubleshooting purposes. This workflow covers audit logging, retention, analysis, and reporting.

## Prerequisites

- WHMCS installation (version 8.x)
- Database access for audit tables
- Log aggregation system (optional)
- Security monitoring tools
- Compliance requirements defined

## Workflow Steps

### Step 1: Audit Trail Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Audit Trail Architecture                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                   Event Sources                       │  │
│  │  • WHMCS Application Events                          │  │
│  │  • Admin Actions                                     │  │
│  │  • API Calls                                         │  │
│  │  • Database Changes                                  │  │
│  │  • File System Changes                               │  │
│  └──────────────────────────────────────────────────────┘  │
│                            │                               │
│                            ▼                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                   Audit Collector                     │  │
│  │  • Hook-based event capture                          │  │
│  │  • Database triggers                                 │  │
│  │  • File integrity monitoring                         │  │
│  └──────────────────────────────────────────────────────┘  │
│                            │                               │
│                            ▼                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                   Audit Storage                      │  │
│  │  • Primary: WHMCS audit_log table                   │  │
│  │  • Secondary: SIEM integration                      │  │
│  │  • Retention: Configurable (default 90 days)        │  │
│  └──────────────────────────────────────────────────────┘  │
│                            │                               │
│                            ▼                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                   Analysis & Reporting               │  │
│  │  • Real-time alerts                                  │  │
│  │  • Compliance reports                                │  │
│  │  • Forensic analysis                                 │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Step 2: Audit Log Database Schema

```sql
-- Create audit log table
CREATE TABLE IF NOT EXISTS `mod_audit_log` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `timestamp` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `user_id` INT UNSIGNED NULL,
    `user_type` ENUM('admin', 'client', 'api', 'system') NOT NULL,
    `action` VARCHAR(100) NOT NULL,
    `category` VARCHAR(50) NOT NULL,
    `resource_type` VARCHAR(50) NULL,
    `resource_id` VARCHAR(100) NULL,
    `ip_address` VARCHAR(45) NULL,
    `user_agent` VARCHAR(500) NULL,
    `old_values` JSON NULL,
    `new_values` JSON NULL,
    `result` ENUM('success', 'failure', 'partial') NOT NULL DEFAULT 'success',
    `session_id` VARCHAR(100) NULL,
    `request_id` VARCHAR(100) NULL,
    `additional_data` JSON NULL,
    PRIMARY KEY (`id`),
    INDEX `idx_timestamp` (`timestamp`),
    INDEX `idx_user` (`user_id`, `user_type`),
    INDEX `idx_action` (`action`),
    INDEX `idx_category` (`category`),
    INDEX `idx_resource` (`resource_type`, `resource_id`),
    INDEX `idx_result` (`result`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Create audit configuration table
CREATE TABLE IF NOT EXISTS `mod_audit_config` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `category` VARCHAR(50) NOT NULL,
    `enabled` TINYINT(1) NOT NULL DEFAULT 1,
    `log_old_values` TINYINT(1) NOT NULL DEFAULT 0,
    `log_new_values` TINYINT(1) NOT NULL DEFAULT 1,
    `retention_days` INT NOT NULL DEFAULT 90,
    `alert_threshold` INT NULL,
    PRIMARY KEY (`id`),
    UNIQUE INDEX `idx_category` (`category`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Insert default configurations
INSERT INTO `mod_audit_config` (`category`, `enabled`, `log_old_values`, `log_new_values`, `retention_days`) VALUES
('authentication', 1, 0, 0, 90),
('authorization', 1, 0, 0, 180),
('data_access', 1, 0, 1, 90),
('data_modification', 1, 1, 1, 365),
('configuration', 1, 1, 1, 365),
('payment', 1, 0, 1, 2555),
('admin_action', 1, 0, 1, 180),
('api_call', 1, 0, 0, 90),
('security_event', 1, 1, 1, 2555);
```

### Step 3: Audit Hook Implementation

```php
<?php
// /var/www/whmcs/includes/hooks/audit_hook.php
// Comprehensive audit logging hook

use WHMCS\Database\Capsule;

class WHMCSAuditLogger {
    private static $instance = null;
    private $pdo;
    private $sessionId;
    private $requestId;
    
    public static function getInstance(): self {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }
    
    private function __construct() {
        $this->pdo = Capsule::connection()->getPdo();
        $this->sessionId = session_id() ?? 'cli';
        $this->requestId = uniqid('req_', true);
    }
    
    public function log(
        string $action,
        string $category,
        ?int $userId = null,
        ?string $userType = null,
        ?string $resourceType = null,
        ?string $resourceId = null,
        ?array $oldValues = null,
        ?array $newValues = null,
        string $result = 'success',
        ?array $additionalData = null
    ): void {
        try {
            $stmt = $this->pdo->prepare("
                INSERT INTO mod_audit_log (
                    user_id, user_type, action, category,
                    resource_type, resource_id,
                    ip_address, user_agent,
                    old_values, new_values,
                    result, session_id, request_id, additional_data
                ) VALUES (
                    ?, ?, ?, ?,
                    ?, ?,
                    ?, ?,
                    ?, ?,
                    ?, ?, ?, ?
                )
            ");
            
            $stmt->execute([
                $userId,
                $userType ?? $this->detectUserType(),
                $action,
                $category,
                $resourceType,
                $resourceId,
                $_SERVER['REMOTE_ADDR'] ?? null,
                substr($_SERVER['HTTP_USER_AGENT'] ?? '', 0, 500),
                $oldValues ? json_encode($oldValues) : null,
                $newValues ? json_encode($newValues) : null,
                $result,
                $this->sessionId,
                $this->requestId,
                $additionalData ? json_encode($additionalData) : null
            ]);
            
            // Check for alert threshold
            $this->checkAlertThreshold($category, $action);
            
        } catch (Exception $e) {
            // Don't let audit logging failure affect main operation
            logActivity("Audit log failed: " . $e->getMessage());
        }
    }
    
    private function detectUserType(): string {
        if (defined('ADMINAREA')) {
            return 'admin';
        }
        if (Session::get('uid')) {
            return 'client';
        }
        return 'api';
    }
    
    private function checkAlertThreshold(string $category, string $action): void {
        $config = Capsule::table('mod_audit_config')
            ->where('category', $category)
            ->first();
        
        if ($config && $config->alert_threshold) {
            $recentCount = Capsule::table('mod_audit_log')
                ->where('category', $category)
                ->where('action', $action)
                ->where('timestamp', '>', date('Y-m-d H:i:s', strtotime('-1 hour')))
                ->count();
            
            if ($recentCount >= $config->alert_threshold) {
                $this->sendSecurityAlert($category, $action, $recentCount);
            }
        }
    }
    
    private function sendSecurityAlert(string $category, string $action, int $count): void {
        // Send alert to security team
        logActivity("SECURITY ALERT: $count $category/$action events in the last hour");
    }
}

// Initialize and register hooks
add_hook('AdminLogin', 1, function($vars) {
    WHMCSAuditLogger::getInstance()->log(
        'admin_login',
        'authentication',
        $vars['adminid'],
        'admin',
        null, null, null, null, null,
        'success',
        ['location' => $vars['ip']]
    );
});

add_hook('AdminLogout', 1, function($vars) {
    WHMCSAuditLogger::getInstance()->log(
        'admin_logout',
        'authentication',
        $vars['adminid'],
        'admin'
    );
});

add_hook('AdminTwoFactorVerify', 1, function($vars) {
    WHMCSAuditLogger::getInstance()->log(
        'admin_2fa_verify',
        'authentication',
        $vars['adminid'],
        'admin',
        null, null, null, null, null,
        $vars['success'] ? 'success' : 'failure'
    );
});

// Client authentication
add_hook('ClientLogin', 1, function($vars) {
    WHMCSAuditLogger::getInstance()->log(
        'client_login',
        'authentication',
        $vars['userid'],
        'client'
    );
});

add_hook('ClientLogout', 1, function($vars) {
    WHMCSAuditLogger::getInstance()->log(
        'client_logout',
        'authentication',
        $vars['userid'],
        'client'
    );
});

// Data modifications
add_hook('ClientEdit', 1, function($vars) {
    WHMCSAuditLogger::getInstance()->log(
        'client_edit',
        'data_modification',
        $_SESSION['adminid'] ?? null,
        'admin',
        'client',
        $vars['userid'],
        $vars['old_values'] ?? null,
        $vars['new_values'] ?? null
    );
});

// Order events
add_hook('OrderCreated', 1, function($vars) {
    WHMCSAuditLogger::getInstance()->log(
        'order_created',
        'data_modification',
        $vars['userid'],
        'client',
        'order',
        $vars['orderid'],
        null,
        ['total' => $vars['total']]
    );
});

// Payment events
add_hook('InvoicePaid', 1, function($vars) {
    WHMCSAuditLogger::getInstance()->log(
        'invoice_paid',
        'payment',
        null, 'system',
        'invoice',
        $vars['invoiceid'],
        null,
        ['amount' => $vars['amount'], 'method' => $vars['paymentmethod']]
    );
});

// Admin actions
add_hook('AdminAreaPage', 1, function($vars) {
    // Log significant admin actions
    $action = $_GET['action'] ?? $_POST['action'] ?? '';
    if (in_array($action, ['moduleupdate', 'configoption', 'delete', 'setup'])) {
        WHMCSAuditLogger::getInstance()->log(
            'admin_config_change',
            'configuration',
            $_SESSION['adminid'],
            'admin',
            'config',
            $action,
            null,
            ['url' => $_SERVER['REQUEST_URI']]
        );
    }
});

// Security events
add_hook('FailedLoginAttempt', 1, function($vars) {
    WHMCSAuditLogger::getInstance()->log(
        'login_failed',
        'security_event',
        null, 'client',
        'client',
        $vars['email'],
        null, null,
        'failure',
        ['ip' => $vars['ip'], 'reason' => $vars['reason']]
    );
});

add_hook('AfterModuleSuspend', 1, function($vars) {
    WHMCSAuditLogger::getInstance()->log(
        'service_suspended',
        'data_modification',
        $_SESSION['adminid'] ?? null,
        'admin',
        'service',
        $vars['serviceid']
    );
});
```

### Step 4: Database Audit Triggers

```sql
-- Create audit triggers for sensitive tables

DELIMITER //

-- Client table audit trigger
CREATE TRIGGER tr_client_audit_update
AFTER UPDATE ON tblclients
FOR EACH ROW
BEGIN
    INSERT INTO mod_audit_log (
        user_id, user_type, action, category,
        resource_type, resource_id,
        ip_address, old_values, new_values, result
    )
    SELECT 
        @admin_id, 'admin', 'client_update', 'data_modification',
        'client', NEW.id,
        @admin_ip,
        JSON_OBJECT('email', OLD.email, 'firstname', OLD.firstname, 'lastname', OLD.lastname, 'status', OLD.status),
        JSON_OBJECT('email', NEW.email, 'firstname', NEW.firstname, 'lastname', NEW.lastname, 'status', NEW.status),
        'success'
    FROM (SELECT @admin_id := NULL, @admin_ip := NULL) AS vars;
END//

-- Invoice audit trigger
CREATE TRIGGER tr_invoice_audit_update
AFTER UPDATE ON tblinvoices
FOR EACH ROW
BEGIN
    IF OLD.status != NEW.status OR OLD.total != NEW.total THEN
        INSERT INTO mod_audit_log (
            user_id, user_type, action, category,
            resource_type, resource_id,
            ip_address, old_values, new_values, result
        )
        SELECT 
            @admin_id, 'admin', 'invoice_status_change', 'payment',
            'invoice', NEW.id,
            @admin_ip,
            JSON_OBJECT('status', OLD.status, 'total', OLD.total),
            JSON_OBJECT('status', NEW.status, 'total', NEW.total),
            'success';
    END IF;
END//

-- Admin user audit trigger
CREATE TRIGGER tr_admin_audit_update
AFTER UPDATE ON tbladmins
FOR EACH ROW
BEGIN
    INSERT INTO mod_audit_log (
        user_id, user_type, action, category,
        resource_type, resource_id,
        ip_address, old_values, new_values, result
    )
    SELECT 
        @admin_id, 'admin', 'admin_user_update', 'authorization',
        'admin', NEW.id,
        @admin_ip,
        JSON_OBJECT('username', OLD.username, 'roleid', OLD.roleid, 'disabled', OLD.disabled),
        JSON_OBJECT('username', NEW.username, 'roleid', NEW.roleid, 'disabled', NEW.disabled),
        'success';
END//

DELIMITER ;
```

### Step 5: Audit Query and Analysis

```php
<?php
// /opt/scripts/audit_analysis.php

class AuditAnalyzer {
    private $pdo;
    
    public function __construct() {
        $this->pdo = new PDO('mysql:host=localhost', 'whmcs', 'password');
    }
    
    public function getFailedLoginAttempts(string $timeRange = '24 hours'): array {
        $stmt = $this->pdo->prepare("
            SELECT 
                ip_address,
                COUNT(*) as attempts,
                GROUP_CONCAT(DISTINCT user_id) as affected_users
            FROM mod_audit_log
            WHERE action = 'login_failed'
            AND timestamp > DATE_SUB(NOW(), INTERVAL ?)
            GROUP BY ip_address
            HAVING attempts >= 3
            ORDER BY attempts DESC
        ");
        
        $stmt->execute([$timeRange]);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    public function getAdminActivitySummary(int $adminId, string $timeRange = '7 days'): array {
        $stmt = $this->pdo->prepare("
            SELECT 
                DATE(timestamp) as date,
                COUNT(*) as total_actions,
                SUM(CASE WHEN result = 'failure' THEN 1 ELSE 0 END) as failures,
                GROUP_CONCAT(DISTINCT category) as categories
            FROM mod_audit_log
            WHERE user_id = ?
            AND user_type = 'admin'
            AND timestamp > DATE_SUB(NOW(), INTERVAL ?)
            GROUP BY DATE(timestamp)
            ORDER BY date DESC
        ");
        
        $stmt->execute([$adminId, $timeRange]);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    public function getDataChanges(array $resourceTypes, string $timeRange = '24 hours'): array {
        $placeholders = implode(',', array_fill(0, count($resourceTypes), '?'));
        
        $stmt = $this->pdo->prepare("
            SELECT 
                timestamp, user_id, user_type,
                action, category,
                resource_type, resource_id,
                old_values, new_values
            FROM mod_audit_log
            WHERE category = 'data_modification'
            AND resource_type IN ($placeholders)
            AND timestamp > DATE_SUB(NOW(), INTERVAL ?)
            ORDER BY timestamp DESC
        ");
        
        $params = array_merge($resourceTypes, [$timeRange]);
        $stmt->execute($params);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    public function getUserAccessPattern(int $userId): array {
        $stmt = $this->pdo->prepare("
            SELECT 
                HOUR(timestamp) as hour,
                DAYOFWEEK(timestamp) as day_of_week,
                COUNT(*) as access_count,
                GROUP_CONCAT(DISTINCT action) as actions
            FROM mod_audit_log
            WHERE user_id = ?
            AND timestamp > DATE_SUB(NOW(), INTERVAL 30 DAY)
            GROUP BY HOUR(timestamp), DAYOFWEEK(timestamp)
            ORDER BY access_count DESC
        ");
        
        $stmt->execute([$userId]);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    public function getSuspiciousActivity(): array {
        return $this->pdo->query("
            SELECT 
                user_id, user_type, ip_address,
                COUNT(*) as event_count,
                MAX(timestamp) as last_event,
                GROUP_CONCAT(DISTINCT action) as actions
            FROM mod_audit_log
            WHERE timestamp > DATE_SUB(NOW(), INTERVAL 1 HOUR)
            AND result = 'failure'
            GROUP BY user_id, user_type, ip_address
            HAVING event_count >= 5
        ")->fetchAll(PDO::FETCH_ASSOC);
    }
    
    public function generateComplianceReport(string $startDate, string $endDate): array {
        $stmt = $this->pdo->prepare("
            SELECT 
                category,
                COUNT(*) as total_events,
                SUM(CASE WHEN result = 'success' THEN 1 ELSE 0 END) as successful,
                SUM(CASE WHEN result = 'failure' THEN 1 ELSE 0 END) as failed,
                COUNT(DISTINCT user_id) as unique_users,
                COUNT(DISTINCT ip_address) as unique_ips
            FROM mod_audit_log
            WHERE timestamp BETWEEN ? AND ?
            GROUP BY category
            ORDER BY total_events DESC
        ");
        
        $stmt->execute([$startDate, $endDate]);
        return [
            'period' => ['start' => $startDate, 'end' => $endDate],
            'categories' => $stmt->fetchAll(PDO::FETCH_ASSOC),
            'total_events' => $this->countEvents($startDate, $endDate)
        ];
    }
    
    private function countEvents(string $start, string $end): int {
        $stmt = $this->pdo->prepare("
            SELECT COUNT(*) FROM mod_audit_log
            WHERE timestamp BETWEEN ? AND ?
        ");
        $stmt->execute([$start, $end]);
        return (int) $stmt->fetchColumn();
    }
}
```

### Step 6: Audit Log Retention

```bash
#!/bin/bash
# /opt/scripts/audit_retention.sh

RETAIN_DAYS=90
ARCHIVE_DAYS=365
LOG_FILE="/var/log/audit_retention.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> "$LOG_FILE"
}

log "Starting audit log retention management"

# Archive old high-security events (before retention)
mysql -u root -p <<EOF
-- Move security events to archive table
INSERT INTO mod_audit_log_archive 
SELECT * FROM mod_audit_log 
WHERE category = 'security_event' 
AND timestamp < DATE_SUB(NOW(), INTERVAL $RETAIN_DAYS DAY)
AND timestamp > DATE_SUB(NOW(), INTERVAL $ARCHIVE_DAYS DAY);

DELETE FROM mod_audit_log 
WHERE category = 'security_event' 
AND timestamp < DATE_SUB(NOW(), INTERVAL $RETAIN_DAYS DAY)
AND timestamp > DATE_SUB(NOW(), INTERVAL $ARCHIVE_DAYS DAY);
EOF

# Delete events past archive period
mysql -u root -p <<EOF
DELETE FROM mod_audit_log 
WHERE timestamp < DATE_SUB(NOW(), INTERVAL $ARCHIVE_DAYS DAY);
EOF

# Optimize table after deletion
mysql -u root -p -e "OPTIMIZE TABLE mod_audit_log;"

# Create retention report
ARCHIVED=$(mysql -u root -p -N -e "SELECT COUNT(*) FROM mod_audit_log_archive")
REMAINING=$(mysql -u root -p -N -e "SELECT COUNT(*) FROM mod_audit_log")

log "Retention complete. Archived: $ARCHIVED, Remaining: $REMAINING"

# Compress and ship old archives to cold storage
aws s3 sync /var/backups/audit_archive/ s3://company-audit-archive/ --storage-class GLACIER
```

## Audit Event Categories

| Category | Events | Retention | Alerts |
|----------|--------|-----------|--------|
| authentication | login, logout, 2FA | 90 days | 5 failures/hour |
| authorization | role change, permission | 180 days | immediate |
| data_access | view, export | 90 days | none |
| data_modification | create, update, delete | 365 days | sensitive fields |
| configuration | settings change | 365 days | immediate |
| payment | charge, refund, dispute | 7 years | immediate |
| admin_action | system config | 180 days | none |
| api_call | api requests | 90 days | errors only |
| security_event | breach, anomaly | 7 years | immediate |

## Compliance Mapping

| Standard | Requirements | WHMCS Implementation |
|----------|--------------|----------------------|
| PCI-DSS | Access logging, audit trails | Audit module + log analysis |
| GDPR | Data access, changes, consent | Client activity logging |
| SOC 2 | User activity, security events | Comprehensive audit trail |
| HIPAA | Access to PHI, changes | Role-based audit logging |

## Best Practices

1. **Log everything**: Capture all significant events
2. **Protect logs**: Ensure audit logs can't be tampered with
3. **Retain appropriately**: Match retention to compliance needs
4. **Monitor actively**: Set up alerts for suspicious activity
5. **Review regularly**: Periodic audit of audit logs
6. **Document changes**: Track who changed what when

## Common Pitfalls

- **Incomplete logging**: Missing critical events
- **No retention**: Logs deleted too soon
- **Unprotected logs**: Logs can be modified
- **Too much noise**: Logging everything without analysis
- **Slow queries**: Audit queries impacting performance

## Verification Checklist

- [ ] Audit logging enabled for all categories
- [ ] Database triggers created
- [ ] Hook-based logging active
- [ ] Retention policies configured
- [ ] Archive process working
- [ ] Alerts configured for security events
- [ ] Compliance reports generation working
- [ ] Log integrity monitoring active

## Related Documentation

- [WHMCS Security Audit](whmcs-security-audit.md)
- [WHMCS Compliance Reporting](whmcs-compliance-reporting.md)
- [WHMCS Incident Response](whmcs-incident-response.md)