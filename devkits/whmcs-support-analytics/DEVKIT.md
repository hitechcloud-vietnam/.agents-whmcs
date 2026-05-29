# WHMCS Support Analytics Module - DEVKIT

## Module Information
- **Name**: Support Analytics
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: Comprehensive support ticket analytics and reporting for WHMCS

## File Structure
```
whmcs-support-analytics/
├── DEVKIT.md
├── hooks.php
├── includes/
│   └── AnalyticsTracker.php
├── templates/
│   ├── admin/
│   │   └── analytics.tpl
│   └── client/
│       └── dashboard.tpl
└── assets/
    └── js/
        └── charts.js
```

## Installation
1. Copy module to `/modules/addons/support_analytics/`
2. Activate via WHMCS Admin > Configuration > Module Settings
3. Configure API keys and preferences

## Core Files

### hooks.php
```php
<?php
/**
 * WHMCS Support Analytics Module
 * 
 * @package WHMCS
 * @copyright Copyright (c) 2024
 * @license Commercial License
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Hook: AdminAreaPage
 */
add_hook('AdminAreaPage', 1, function($vars) {
    $routeUri = $vars['routeUri'] ?? '';
    
    if (strpos($routeUri, '/support/') !== false) {
        return [
            'analyticsWidget' => [
                'display' => true,
                'template' => 'analytics:admin/widgets/support-summary'
            ]
        ];
    }
    
    return [];
});

/**
 * Hook: TicketCreated
 */
add_hook('TicketOpen', 1, function($vars) {
    $ticketId = $vars['ticketid'] ?? 0;
    
    AnalyticsTracker::trackEvent('ticket_created', [
        'ticket_id' => $ticketId,
        'department_id' => $vars['deptid'] ?? 0,
        'priority' => $vars['priority'] ?? 'Medium',
        'timestamp' => date('Y-m-d H:i:s')
    ]);
    
    return [];
});

/**
 * Hook: TicketReplied
 */
add_hook('TicketReply', 1, function($vars) {
    AnalyticsTracker::trackEvent('ticket_replied', [
        'ticket_id' => $vars['ticketid'] ?? 0,
        'response_time' => $vars['responsetime'] ?? 0,
        'timestamp' => date('Y-m-d H:i:s')
    ]);
    
    return [];
});

/**
 * Hook: TicketClosed
 */
add_hook('TicketClose', 1, function($vars) {
    AnalyticsTracker::trackEvent('ticket_closed', [
        'ticket_id' => $vars['ticketid'] ?? 0,
        'resolution_time' => $vars['resolutiontime'] ?? 0,
        'client_rating' => $vars['rating'] ?? null,
        'timestamp' => date('Y-m-d H:i:s')
    ]);
    
    return [];
});

/**
 * Hook: AdminHomeDashboard
 */
add_hook('AdminHomeDashboard', 1, function($vars) {
    return [
        'supportMetrics' => AnalyticsTracker::getDashboardMetrics()
    ];
});

/**
 * Hook: ClientAreaPage
 */
add_hook('ClientAreaPage', 1, function($vars) {
    $userId = $_SESSION['uid'] ?? 0;
    
    return [
        'clientSupportStats' => AnalyticsTracker::getClientStats($userId)
    ];
});
```

### includes/AnalyticsTracker.php
```php
<?php
/**
 * Analytics Tracker Class
 * 
 * Handles tracking and reporting of support metrics
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class AnalyticsTracker
{
    private static $dbTable = 'mod_support_analytics';
    private static $cacheExpiry = 300; // 5 minutes
    
    /**
     * Track a support event
     */
    public static function trackEvent($eventType, $data)
    {
        $data = json_encode([
            'event_type' => $eventType,
            'event_data' => $data,
            'created_at' => date('Y-m-d H:i:s'),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'unknown'
        ]);
        
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$dbTable . " 
            (event_type, event_data, created_at, ip_address)
            VALUES ('" . db_escape_string($eventType) . "', '" . db_escape_string($data) . "', NOW(), '" . db_escape_string($_SERVER['REMOTE_ADDR'] ?? '') . "')
        ");
        
        self::invalidateCache();
    }
    
    /**
     * Get dashboard metrics
     */
    public static function getDashboardMetrics()
    {
        $cacheKey = 'dashboard_metrics';
        $cached = self::getCached($cacheKey);
        
        if ($cached) {
            return $cached;
        }
        
        $result = full_query("
            SELECT 
                COUNT(*) as total_tickets,
                SUM(CASE WHEN status = 'Open' THEN 1 ELSE 0 END) as open_tickets,
                SUM(CASE WHEN status = 'Closed' THEN 1 ELSE 0 END) as closed_tickets,
                AVG(response_time) as avg_response_time,
                AVG(resolution_time) as avg_resolution_time
            FROM " . TABLE_PREFIX . "tbltickets
            WHERE created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
        ");
        
        $data = mysql_fetch_array($result);
        
        self::setCached($cacheKey, $data);
        
        return $data;
    }
    
    /**
     * Get client-specific support stats
     */
    public static function getClientStats($clientId)
    {
        if (!$clientId) {
            return null;
        }
        
        $result = full_query("
            SELECT 
                COUNT(*) as total_tickets,
                SUM(CASE WHEN status = 'Open' THEN 1 ELSE 0 END) as open_tickets,
                SUM(CASE WHEN status = 'Answered' THEN 1 ELSE 0 END) as answered_tickets,
                AVG(response_time) as avg_response_time
            FROM " . TABLE_PREFIX . "tbltickets
            WHERE userid = " . (int)$clientId
        );
        
        return mysql_fetch_array($result);
    }
    
    /**
     * Get ticket trend data
     */
    public static function getTicketTrends($days = 30)
    {
        $result = full_query("
            SELECT 
                DATE(created_at) as date,
                COUNT(*) as ticket_count,
                SUM(CASE WHEN status = 'Open' THEN 1 ELSE 0 END) as open,
                SUM(CASE WHEN status = 'Closed' THEN 1 ELSE 0 END) as closed
            FROM " . TABLE_PREFIX . "tbltickets
            WHERE created_at >= DATE_SUB(NOW(), INTERVAL " . (int)$days . " DAY)
            GROUP BY DATE(created_at)
            ORDER BY date ASC
        ");
        
        $trends = [];
        while ($row = mysql_fetch_array($result)) {
            $trends[] = $row;
        }
        
        return $trends;
    }
    
    /**
     * Get department performance metrics
     */
    public static function getDepartmentMetrics()
    {
        $result = full_query("
            SELECT 
                d.name as department,
                COUNT(t.id) as total_tickets,
                AVG(TIMESTAMPDIFF(HOUR, t.created_at, t.last_reply)) as avg_first_response_hours,
                SUM(CASE WHEN t.status = 'Closed' THEN 1 ELSE 0 END) as resolved,
                SUM(CASE WHEN t.status = 'Open' THEN 1 ELSE 0 END) as pending
            FROM " . TABLE_PREFIX . "tblticketdepartments d
            LEFT JOIN " . TABLE_PREFIX . "tbltickets t ON d.id = t.deptid
            GROUP BY d.id, d.name
            ORDER BY total_tickets DESC
        ");
        
        $metrics = [];
        while ($row = mysql_fetch_array($result)) {
            $metrics[] = $row;
        }
        
        return $metrics;
    }
    
    /**
     * Get staff performance data
     */
    public static function getStaffPerformance($staffId = null)
    {
        $whereClause = $staffId ? "WHERE t.admin = " . (int)$staffId : "";
        
        $result = full_query("
            SELECT 
                a.id,
                a.username,
                COUNT(t.id) as tickets_handled,
                AVG(TIMESTAMPDIFF(HOUR, t.created_at, t.last_reply)) as avg_response_time,
                SUM(CASE WHEN t.status = 'Closed' THEN 1 ELSE 0 END) as resolved
            FROM " . TABLE_PREFIX . "tbladmins a
            LEFT JOIN " . TABLE_PREFIX . "tbltickets t ON a.id = t.admin " . $whereClause . "
            GROUP BY a.id, a.username
            ORDER BY tickets_handled DESC
        ");
        
        $performance = [];
        while ($row = mysql_fetch_array($result)) {
            $performance[] = $row;
        }
        
        return $performance;
    }
    
    /**
     * Get CSAT (Customer Satisfaction) data
     */
    public static function getCSATData()
    {
        $result = full_query("
            SELECT 
                AVG(rating) as avg_rating,
                COUNT(*) as total_ratings,
                SUM(CASE WHEN rating >= 4 THEN 1 ELSE 0 END) as positive,
                SUM(CASE WHEN rating <= 2 THEN 1 ELSE 0 END) as negative
            FROM " . TABLE_PREFIX . "tbltickets
            WHERE rating IS NOT NULL
            AND created_at >= DATE_SUB(NOW(), INTERVAL 90 DAY)
        ");
        
        return mysql_fetch_array($result);
    }
    
    private static function getCached($key)
    {
        $result = full_query("
            SELECT cache_data FROM " . TABLE_PREFIX . "mod_support_analytics_cache
            WHERE cache_key = '" . db_escape_string($key) . "'
            AND created_at > DATE_SUB(NOW(), INTERVAL " . self::$cacheExpiry . " SECOND)
        ");
        
        $row = mysql_fetch_array($result);
        return $row ? json_decode($row['cache_data'], true) : null;
    }
    
    private static function setCached($key, $data)
    {
        full_query("
            INSERT INTO " . TABLE_PREFIX . "mod_support_analytics_cache (cache_key, cache_data, created_at)
            VALUES ('" . db_escape_string($key) . "', '" . db_escape_string(json_encode($data)) . "', NOW())
            ON DUPLICATE KEY UPDATE cache_data = VALUES(cache_data), created_at = NOW()
        ");
    }
    
    private static function invalidateCache()
    {
        // Cache invalidation logic
    }
}
```

### Activation/Deactivation Functions
```php
/**
 * Activate module
 */
function support_analytics_activate()
{
    // Create database tables
    $query = "
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_support_analytics (
            id INT AUTO_INCREMENT PRIMARY KEY,
            event_type VARCHAR(100) NOT NULL,
            event_data TEXT,
            created_at DATETIME NOT NULL,
            ip_address VARCHAR(45)
        )
    ";
    full_query($query);
    
    $query = "
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_support_analytics_cache (
            cache_key VARCHAR(255) PRIMARY KEY,
            cache_data TEXT,
            created_at DATETIME NOT NULL
        )
    ";
    full_query($query);
    
    // Create indexes
    full_query("CREATE INDEX idx_event_type ON " . TABLE_PREFIX . "mod_support_analytics(event_type)");
    full_query("CREATE INDEX idx_created_at ON " . TABLE_PREFIX . "mod_support_analytics(created_at)");
    
    return [
        'status' => 'success',
        'description' => 'Support Analytics module activated successfully'
    ];
}

/**
 * Deactivate module
 */
function support_analytics_deactivate()
{
    // Optional: Keep data on deactivation
    // To delete data, uncomment below:
    // full_query("DROP TABLE IF EXISTS " . TABLE_PREFIX . "mod_support_analytics");
    // full_query("DROP TABLE IF EXISTS " . TABLE_PREFIX . "mod_support_analytics_cache");
    
    return [
        'status' => 'success',
        'description' => 'Support Analytics module deactivated'
    ];
}
```

## Configuration
```php
/**
 * Module configuration
 */
function support_analytics_config()
{
    return [
        'name' => 'Support Analytics',
        'description' => 'Comprehensive support ticket analytics and reporting',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'analytics_enabled' => [
                'Type' => 'yesno',
                'FriendlyName' => 'Enable Analytics',
                'Description' => 'Enable real-time analytics tracking'
            ],
            'retention_days' => [
                'Type' => 'text',
                'FriendlyName' => 'Data Retention (Days)',
                'Default' => '90',
                'Description' => 'Number of days to keep analytics data'
            ],
            'cache_expiry' => [
                'Type' => 'dropdown',
                'FriendlyName' => 'Cache Expiry',
                'Options' => [
                    '60' => '1 minute',
                    '300' => '5 minutes',
                    '600' => '10 minutes',
                    '1800' => '30 minutes'
                ],
                'Default' => '300'
            ]
        ]
    ];
}
```

## Usage Examples
```php
// Track custom events
AnalyticsTracker::trackEvent('custom_action', [
    'user_id' => 123,
    'action' => 'viewed_report',
    'metadata' => ['report_type' => 'weekly']
]);

// Get dashboard data
$metrics = AnalyticsTracker::getDashboardMetrics();

// Get trends
$trends = AnalyticsTracker::getTicketTrends(30);

// Get CSAT data
$satisfaction = AnalyticsTracker::getCSATData();
```