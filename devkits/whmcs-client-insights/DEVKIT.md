# WHMCS Client Insights Module - DEVKIT

## Module Information
- **Name**: Client Insights
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: Deep client behavior analytics and insights for WHMCS

## Installation
1. Copy module to `/modules/addons/client_insights/`
2. Activate via WHMCS Admin > Configuration > Module Settings

## Core Files

### hooks.php
```php
<?php
/**
 * WHMCS Client Insights Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('ClientAreaPage', 1, function($vars) {
    $userId = $_SESSION['uid'] ?? 0;
    if ($userId) {
        ClientInsights::trackEngagement($userId);
    }
    return [];
});

add_hook('OrderPlaced', 1, function($vars) {
    ClientInsights::trackOrder($vars);
    return [];
});

add_hook('InvoicePaid', 1, function($vars) {
    ClientInsights::trackPayment($vars);
    return [];
});

add_hook('ServiceCreated', 1, function($vars) {
    ClientInsights::trackServiceActivity($vars);
    return [];
});
```

### includes/ClientInsights.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class ClientInsights
{
    private static $table = 'mod_client_insights';
    
    public static function trackEngagement($clientId)
    {
        $data = [
            'client_id' => $clientId,
            'page_views' => 1,
            'last_activity' => date('Y-m-d H:i:s'),
            'session_data' => json_encode($_SESSION)
        ];
        
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$table . " (client_id, page_views, last_activity, session_data, created_at)
            VALUES (" . (int)$clientId . ", 1, NOW(), '" . db_escape_string($data['session_data']) . "', NOW())
            ON DUPLICATE KEY UPDATE 
                page_views = page_views + 1,
                last_activity = NOW(),
                session_data = VALUES(session_data)
        ");
    }
    
    public static function trackOrder($vars)
    {
        full_query("
            INSERT INTO " . TABLE_PREFIX . "mod_client_insights_events 
            (client_id, event_type, event_data, created_at)
            VALUES (" . (int)($vars['userid'] ?? 0) . ", 'order_placed', '" . db_escape_string(json_encode($vars)) . "', NOW())
        ");
    }
    
    public static function trackPayment($vars)
    {
        full_query("
            INSERT INTO " . TABLE_PREFIX . "mod_client_insights_events 
            (client_id, event_type, event_data, created_at)
            VALUES (" . (int)($vars['userid'] ?? 0) . ", 'invoice_paid', '" . db_escape_string(json_encode($vars)) . "', NOW())
        ");
    }
    
    public static function trackServiceActivity($vars)
    {
        full_query("
            INSERT INTO " . TABLE_PREFIX . "mod_client_insights_events 
            (client_id, event_type, event_data, created_at)
            VALUES (" . (int)($vars['userid'] ?? 0) . ", 'service_created', '" . db_escape_string(json_encode($vars)) . "', NOW())
        ");
    }
    
    public static function getClientProfile($clientId)
    {
        $result = full_query("
            SELECT 
                c.*,
                COUNT(DISTINCT o.id) as total_orders,
                SUM(o.amount) as total_spent,
                COUNT(DISTINCT t.id) as support_tickets,
                AVG(t.response_time) as avg_ticket_response
            FROM " . TABLE_PREFIX . "tblclients c
            LEFT JOIN " . TABLE_PREFIX . "tblorders o ON c.id = o.userid
            LEFT JOIN " . TABLE_PREFIX . "tbltickets t ON c.id = t.userid
            WHERE c.id = " . (int)$clientId . "
            GROUP BY c.id
        ");
        
        return mysql_fetch_array($result);
    }
    
    public static function getLifetimeValue($clientId)
    {
        $result = full_query("
            SELECT 
                SUM(amount) as total_revenue,
                COUNT(DISTINCT service) as services_count,
                COUNT(DISTINCT domain) as domains_count
            FROM " . TABLE_PREFIX . "tblorders
            WHERE userid = " . (int)$clientId . " AND status = 'Active'
        ");
        
        return mysql_fetch_array($result);
    }
    
    public static function getChurnRisk($clientId)
    {
        $data = self::getClientProfile($clientId);
        
        $riskScore = 0;
        
        // Check for inactivity
        if (strtotime($data['lastlogin']) < strtotime('-90 days')) {
            $riskScore += 30;
        }
        
        // Check support intensity
        if ($data['support_tickets'] > 10) {
            $riskScore += 20;
        }
        
        // Check payment issues
        if ($data['failedpayments'] > 0) {
            $riskScore += 25;
        }
        
        return [
            'risk_score' => $riskScore,
            'risk_level' => $riskScore > 50 ? 'High' : ($riskScore > 25 ? 'Medium' : 'Low')
        ];
    }
}

function client_insights_activate()
{
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_client_insights (
            id INT AUTO_INCREMENT PRIMARY KEY,
            client_id INT NOT NULL,
            page_views INT DEFAULT 1,
            last_activity DATETIME,
            session_data TEXT,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_client_insights_events (
            id INT AUTO_INCREMENT PRIMARY KEY,
            client_id INT NOT NULL,
            event_type VARCHAR(50),
            event_data TEXT,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    return ['status' => 'success', 'description' => 'Client Insights activated'];
}

function client_insights_deactivate()
{
    return ['status' => 'success', 'description' => 'Client Insights deactivated'];
}

function client_insights_config()
{
    return [
        'name' => 'Client Insights',
        'description' => 'Deep client behavior analytics and insights',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'tracking_enabled' => [
                'Type' => 'yesno',
                'FriendlyName' => 'Enable Tracking',
                'Default' => '1'
            ],
            'session_timeout' => [
                'Type' => 'text',
                'FriendlyName' => 'Session Timeout (minutes)',
                'Default' => '30'
            ]
        ]
    ];
}
```