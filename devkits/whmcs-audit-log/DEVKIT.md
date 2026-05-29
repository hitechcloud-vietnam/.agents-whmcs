# WHMCS Audit Log Module - DEVKIT

## Module Information
- **Name**: Audit Log
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: Comprehensive audit logging for compliance and security

## Installation
1. Copy to `/modules/addons/audit_log/`
2. Activate via WHMCS Admin

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('AdminAreaPage', 1, function($vars) {
    AuditLog::logAdminAction($vars);
});

add_hook('ClientAreaPage', 1, function($vars) {
    AuditLog::logClientAction($vars);
});

add_hook('AfterCalculateCartTotals', 1, function($vars) {
    AuditLog::logOrderAction($vars);
});
```

### includes/AuditLog.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class AuditLog
{
    private static $table = 'mod_audit_log';
    
    public static function log($action, $entityType, $entityId, $userId, $userType, $details = [], $ipAddress = null)
    {
        $data = json_encode($details);
        
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$table . "
            (action, entity_type, entity_id, user_id, user_type, details, ip_address, created_at)
            VALUES (
                '" . db_escape_string($action) . "',
                '" . db_escape_string($entityType) . "',
                " . (int)$entityId . ",
                " . (int)$userId . ",
                '" . db_escape_string($userType) . "',
                '" . db_escape_string($data) . "',
                '" . db_escape_string($ipAddress ?? ($_SERVER['REMOTE_ADDR'] ?? 'unknown')) . "',
                NOW()
            )
        ");
        
        return mysql_insert_id();
    }
    
    public static function logAdminAction($vars)
    {
        if (empty($_SESSION['adminid'])) return;
        
        $route = $vars['routeUri'] ?? '';
        $action = str_replace('/', '_', trim($route, '/'));
        
        self::log($action, 'admin_area', 0, $_SESSION['adminid'], 'admin', ['page' => $route]);
    }
    
    public static function logClientAction($vars)
    {
        if (empty($_SESSION['uid'])) return;
        
        $route = $vars['routeUri'] ?? '';
        
        self::log('page_view', 'client_area', $_SESSION['uid'], $_SESSION['uid'], 'client', ['page' => $route]);
    }
    
    public static function logOrderAction($vars)
    {
        if (empty($_SESSION['uid'])) return;
        
        self::log('cart_modified', 'cart', 0, $_SESSION['uid'], 'client', ['total' => $vars['total'] ?? 0]);
    }
    
    public static function getLogs($filters = [], $limit = 100)
    {
        $where = ['1=1'];
        
        if (!empty($filters['user_id'])) {
            $where[] = "user_id = " . (int)$filters['user_id'];
        }
        
        if (!empty($filters['user_type'])) {
            $where[] = "user_type = '" . db_escape_string($filters['user_type']) . "'";
        }
        
        if (!empty($filters['action'])) {
            $where[] = "action LIKE '%" . db_escape_string($filters['action']) . "%'";
        }
        
        if (!empty($filters['entity_type'])) {
            $where[] = "entity_type = '" . db_escape_string($filters['entity_type']) . "'";
        }
        
        if (!empty($filters['start_date'])) {
            $where[] = "created_at >= '" . db_escape_string($filters['start_date']) . "'";
        }
        
        if (!empty($filters['end_date'])) {
            $where[] = "created_at <= '" . db_escape_string($filters['end_date']) . "'";
        }
        
        $whereClause = implode(' AND ', $where);
        
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$table . "
            WHERE " . $whereClause . "
            ORDER BY created_at DESC
            LIMIT " . (int)$limit
        ");
        
        $logs = [];
        while ($row = mysql_fetch_array($result)) {
            $row['details'] = json_decode($row['details'], true);
            $logs[] = $row;
        }
        
        return $logs;
    }
    
    public static function getRecentActivity($userId, $limit = 50)
    {
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$table . "
            WHERE user_id = " . (int)$userId . "
            ORDER BY created_at DESC
            LIMIT " . (int)$limit
        ");
        
        $activity = [];
        while ($row = mysql_fetch_array($result)) {
            $activity[] = $row;
        }
        
        return $activity;
    }
    
    public static function exportLogs($filters = [])
    {
        $logs = self::getLogs($filters, 10000);
        
        $csv = "Date,Action,Entity Type,Entity ID,User ID,User Type,IP Address,Details\n";
        
        foreach ($logs as $log) {
            $csv .= '"' . $log['created_at'] . '",';
            $csv .= '"' . $log['action'] . '",';
            $csv .= '"' . $log['entity_type'] . '",';
            $csv .= $log['entity_id'] . ',';
            $csv .= $log['user_id'] . ',';
            $csv .= '"' . $log['user_type'] . '",';
            $csv .= '"' . $log['ip_address'] . '",';
            $csv .= '"' . addslashes($log['details']) . "\"\n";
        }
        
        return $csv;
    }
}

function audit_log_activate()
{
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_audit_log (
            id INT AUTO_INCREMENT PRIMARY KEY,
            action VARCHAR(100) NOT NULL,
            entity_type VARCHAR(50),
            entity_id INT,
            user_id INT NOT NULL,
            user_type VARCHAR(20),
            details TEXT,
            ip_address VARCHAR(45),
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("CREATE INDEX idx_user ON " . TABLE_PREFIX . "mod_audit_log(user_id)");
    full_query("CREATE INDEX idx_action ON " . TABLE_PREFIX . "mod_audit_log(action)");
    full_query("CREATE INDEX idx_created ON " . TABLE_PREFIX . "mod_audit_log(created_at)");
    
    return ['status' => 'success', 'description' => 'Audit Log activated'];
}

function audit_log_deactivate()
{
    return ['status' => 'success', 'description' => 'Audit Log deactivated'];
}

function audit_log_config()
{
    return [
        'name' => 'Audit Log',
        'description' => 'Comprehensive audit logging for compliance',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'retention_days' => ['Type' => 'text', 'FriendlyName' => 'Retention (Days)', 'Default' => '365'],
            'log_admin_pages' => ['Type' => 'yesno', 'FriendlyName' => 'Log Admin Pages', 'Default' => '1'],
            'log_client_pages' => ['Type' => 'yesno', 'FriendlyName' => 'Log Client Pages', 'Default' => '0']
        ]
    ];
}
```