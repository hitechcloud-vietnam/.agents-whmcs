# WHMCS Service Health Monitor - DEVKIT

## Module Information
- **Name**: Service Health Monitor
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: Monitor service uptime and availability

## Installation
1. Copy to `/modules/addons/service_health_monitor/`
2. Activate via WHMCS Admin

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('DailyCronJob', 1, function($vars) {
    ServiceHealthMonitor::runHealthChecks();
});

add_hook('ServiceCreated', 1, function($vars) {
    ServiceHealthMonitor::addServiceToMonitor($vars);
});

add_hook('ServiceTerminated', 1, function($vars) {
    ServiceHealthMonitor::removeServiceFromMonitor($vars['serviceid']);
});
```

### includes/ServiceHealthMonitor.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class ServiceHealthMonitor
{
    private static $table = 'mod_service_health_monitor';
    private static $checksTable = 'mod_health_checks';
    
    public static function addServiceToMonitor($vars)
    {
        $serviceId = $vars['serviceid'] ?? 0;
        $domain = $vars['domain'] ?? '';
        
        if (empty($domain)) return;
        
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$table . "
            (service_id, domain, status, created_at)
            VALUES (
                " . (int)$serviceId . ",
                '" . db_escape_string($domain) . "',
                'unknown',
                NOW()
            )
        ");
    }
    
    public static function removeServiceFromMonitor($serviceId)
    {
        full_query("
            DELETE FROM " . TABLE_PREFIX . self::$table . "
            WHERE service_id = " . (int)$serviceId
        );
    }
    
    public static function runHealthChecks()
    {
        $services = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$table . "
            WHERE monitored = 1
        ");
        
        while ($service = mysql_fetch_array($services)) {
            $result = self::checkServiceHealth($service);
            
            // Update status
            full_query("
                UPDATE " . TABLE_PREFIX . self::$table . "
                SET status = '" . db_escape_string($result['status']) . "',
                    last_check = NOW(),
                    last_response_time = " . (float)$result['response_time'] . "
                WHERE id = " . (int)$service['id']
            );
            
            // Log check
            self::logCheck($service['id'], $result);
            
            // Alert if down
            if ($result['status'] == 'down') {
                self::alertAdmins($service, $result);
            }
        }
    }
    
    private static function checkServiceHealth($service)
    {
        $startTime = microtime(true);
        
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, 'https://' . $service['domain']);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 10);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
        curl_setopt($ch, CURLOPT_FOLLOWLOCATION, true);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);
        
        $endTime = microtime(true);
        $responseTime = round(($endTime - $startTime) * 1000, 2);
        
        $status = 'up';
        if ($error || $httpCode >= 500) {
            $status = 'down';
        } elseif ($httpCode >= 400) {
            $status = 'degraded';
        }
        
        return [
            'status' => $status,
            'http_code' => $httpCode,
            'response_time' => $responseTime,
            'error' => $error
        ];
    }
    
    private static function logCheck($monitorId, $result)
    {
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$checksTable . "
            (monitor_id, status, http_code, response_time, error, checked_at)
            VALUES (
                " . (int)$monitorId . ",
                '" . db_escape_string($result['status']) . "',
                " . (int)($result['http_code'] ?? 0) . ",
                " . (float)$result['response_time'] . ",
                " . ($result['error'] ? "'" . db_escape_string($result['error']) . "'" : "NULL") . ",
                NOW()
            )
        ");
    }
    
    private static function alertAdmins($service, $result)
    {
        // Check if already alerted recently
        $recent = full_query("
            SELECT id FROM " . TABLE_PREFIX . "mod_health_alerts
            WHERE monitor_id = " . (int)$service['id'] . "
            AND created_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)
        ");
        
        if (mysql_fetch_array($recent)) {
            return; // Already alerted
        }
        
        // Log alert
        full_query("
            INSERT INTO " . TABLE_PREFIX . "mod_health_alerts
            (monitor_id, alert_type, message, created_at)
            VALUES (
                " . (int)$service['id'] . ",
                'service_down',
                'Service " . db_escape_string($service['domain']) . " is down: " . db_escape_string($result['error'] ?? '') . "',
                NOW()
            )
        ");
        
        // Send notification
        send_admin_notification(
            'all',
            'Service Down Alert',
            'Service ' . $service['domain'] . ' is not responding. Error: ' . ($result['error'] ?? 'Unknown')
        );
    }
    
    public static function getServiceStatus($serviceId)
    {
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$table . "
            WHERE service_id = " . (int)$serviceId
        ");
        
        return mysql_fetch_array($result);
    }
    
    public static function getUptimeStats($serviceId, $days = 30)
    {
        $monitor = full_query("
            SELECT id FROM " . TABLE_PREFIX . self::$table . "
            WHERE service_id = " . (int)$serviceId
        ");
        $data = mysql_fetch_array($monitor);
        
        if (!$data) return null;
        
        $result = full_query("
            SELECT 
                COUNT(*) as total_checks,
                SUM(CASE WHEN status = 'up' THEN 1 ELSE 0 END) as up_count,
                AVG(response_time) as avg_response_time
            FROM " . TABLE_PREFIX . self::$checksTable . "
            WHERE monitor_id = " . (int)$data['id'] . "
            AND checked_at >= DATE_SUB(NOW(), INTERVAL " . (int)$days . " DAY)
        ");
        
        $stats = mysql_fetch_array($result);
        
        $uptime = $stats['total_checks'] > 0 
            ? round(($stats['up_count'] / $stats['total_checks']) * 100, 2) 
            : 0;
        
        return [
            'uptime_percent' => $uptime,
            'total_checks' => $stats['total_checks'],
            'avg_response_time' => round($stats['avg_response_time'], 2)
        ];
    }
    
    public static function getAllServicesStatus()
    {
        $result = full_query("
            SELECT m.*, h.domain as service_domain
            FROM " . TABLE_PREFIX . self::$table . " m
            JOIN " . TABLE_PREFIX . "tblhosting h ON m.service_id = h.id
            WHERE m.monitored = 1
            ORDER BY m.status ASC, m.last_check DESC
        ");
        
        $services = [];
        while ($row = mysql_fetch_array($result)) {
            $services[] = $row;
        }
        
        return $services;
    }
}

function service_health_monitor_activate()
{
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_service_health_monitor (
            id INT AUTO_INCREMENT PRIMARY KEY,
            service_id INT NOT NULL,
            domain VARCHAR(255) NOT NULL,
            status VARCHAR(20) DEFAULT 'unknown',
            monitored TINYINT(1) DEFAULT 1,
            last_check DATETIME,
            last_response_time DECIMAL(10,2),
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_health_checks (
            id INT AUTO_INCREMENT PRIMARY KEY,
            monitor_id INT NOT NULL,
            status VARCHAR(20),
            http_code INT,
            response_time DECIMAL(10,2),
            error TEXT,
            checked_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_health_alerts (
            id INT AUTO_INCREMENT PRIMARY KEY,
            monitor_id INT NOT NULL,
            alert_type VARCHAR(50),
            message TEXT,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("CREATE INDEX idx_monitor ON " . TABLE_PREFIX . "mod_health_checks(monitor_id)");
    
    return ['status' => 'success', 'description' => 'Service Health Monitor activated'];
}

function service_health_monitor_deactivate()
{
    return ['status' => 'success', 'description' => 'Service Health Monitor deactivated'];
}

function service_health_monitor_config()
{
    return [
        'name' => 'Service Health Monitor',
        'description' => 'Monitor service uptime and availability',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'check_interval' => [
                'Type' => 'dropdown',
                'FriendlyName' => 'Check Interval',
                'Options' => [
                    '5' => 'Every 5 minutes',
                    '15' => 'Every 15 minutes',
                    '30' => 'Every 30 minutes',
                    '60' => 'Every hour'
                ],
                'Default' => '15'
            ],
            'alert_on_down' => [
                'Type' => 'yesno',
                'FriendlyName' => 'Alert on Service Down',
                'Default' => '1'
            ],
            'timeout' => [
                'Type' => 'text',
                'FriendlyName' => 'Request Timeout (seconds)',
                'Default' => '10'
            ]
        ]
    ];
}
```