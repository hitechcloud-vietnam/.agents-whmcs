# WHMCS KPI Dashboard Module - DEVKIT

## Module Information
- **Name**: KPI Dashboard
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: Real-time KPI dashboard with charts and metrics

## Installation
1. Copy to `/modules/addons/kpi_dashboard/`
2. Activate via WHMCS Admin

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('AdminHomepage', 1, function($vars) {
    return ['kpiWidgets' =>KPIDashboard::getHomepageWidgets()];
});

add_hook('AdminAreaPage', 1, function($vars) {
    if (strpos($vars['routeUri'] ?? '', '/kpi') !== false) {
        return ['kpiEnabled' => true];
    }
    return [];
});
```

### includes/KPIDashboard.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class KPIDashboard
{
    private static $metricsTable = 'mod_kpi_metrics';
    
    public static function getHomepageWidgets()
    {
        return [
            'revenue_today' => self::getRevenueToday(),
            'new_orders' => self::getNewOrdersToday(),
            'active_services' => self::getActiveServicesCount(),
            'open_tickets' => self::getOpenTicketsCount(),
            'mrr_growth' => self::getMRRGrowth(),
            'churn_rate' => self::getChurnRate()
        ];
    }
    
    public static function getRevenueToday()
    {
        $result = full_query("
            SELECT COALESCE(SUM(amountin), 0) as revenue
            FROM " . TABLE_PREFIX . "tblaccounts
            WHERE DATE(date) = CURDATE()
        ");
        
        $row = mysql_fetch_array($result);
        return $row['revenue'] ?? 0;
    }
    
    public static function getNewOrdersToday()
    {
        $result = full_query("
            SELECT COUNT(*) as orders
            FROM " . TABLE_PREFIX . "tblorders
            WHERE DATE(date) = CURDATE()
        ");
        
        $row = mysql_fetch_array($result);
        return $row['orders'] ?? 0;
    }
    
    public static function getActiveServicesCount()
    {
        $result = full_query("
            SELECT COUNT(*) as count
            FROM " . TABLE_PREFIX . "tblhosting
            WHERE domainstatus = 'Active'
        ");
        
        $row = mysql_fetch_array($result);
        return $row['count'] ?? 0;
    }
    
    public static function getOpenTicketsCount()
    {
        $result = full_query("
            SELECT COUNT(*) as count
            FROM " . TABLE_PREFIX . "tbltickets
            WHERE status NOT IN ('Closed', 'Resolved')
        ");
        
        $row = mysql_fetch_array($result);
        return $row['count'] ?? 0;
    }
    
    public static function getMRRGrowth()
    {
        $currentMonth = full_query("
            SELECT COALESCE(SUM(monthly), 0) as mrr
            FROM " . TABLE_PREFIX . "tblhosting
            WHERE domainstatus = 'Active'
            AND billingcycle IN ('Monthly', 'Quarterly', 'Annually')
        ");
        
        $currentMRR = mysql_fetch_array($currentMonth)['mrr'] ?? 0;
        
        $lastMonth = full_query("
            SELECT COALESCE(SUM(amount), 0) as mrr
            FROM " . TABLE_PREFIX . "tblorders
            WHERE status = 'Active'
            AND date >= DATE_SUB(CURDATE(), INTERVAL 1 MONTH)
            AND date < CURDATE()
        ");
        
        $lastMRR = mysql_fetch_array($lastMonth)['mrr'] ?? 0;
        
        if ($lastMRR > 0) {
            return round((($currentMRR - $lastMRR) / $lastMRR) * 100, 2);
        }
        
        return 0;
    }
    
    public static function getChurnRate()
    {
        $totalLastMonth = full_query("
            SELECT COUNT(*) as total
            FROM " . TABLE_PREFIX . "tblhosting
            WHERE created_at < DATE_SUB(CURDATE(), INTERVAL 1 MONTH)
        ");
        
        $total = mysql_fetch_array($totalLastMonth)['total'] ?? 1;
        
        $churned = full_query("
            SELECT COUNT(*) as churned
            FROM " . TABLE_PREFIX . "tblhosting
            WHERE domainstatus = 'Terminated'
            AND termination_date >= DATE_SUB(CURDATE(), INTERVAL 1 MONTH)
        ");
        
        $churnedCount = mysql_fetch_array($churned)['churned'] ?? 0;
        
        return round(($churnedCount / $total) * 100, 2);
    }
    
    public static function getRevenueChart($days = 30)
    {
        $result = full_query("
            SELECT 
                DATE(date) as date,
                SUM(amountin) as revenue
            FROM " . TABLE_PREFIX . "tblaccounts
            WHERE date >= DATE_SUB(CURDATE(), INTERVAL " . (int)$days . " DAY)
            GROUP BY DATE(date)
            ORDER BY date ASC
        ");
        
        $data = [];
        while ($row = mysql_fetch_array($result)) {
            $data[] = $row;
        }
        
        return $data;
    }
    
    public static function getOrderChart($days = 30)
    {
        $result = full_query("
            SELECT 
                DATE(date) as date,
                COUNT(*) as orders,
                SUM(amount) as total
            FROM " . TABLE_PREFIX . "tblorders
            WHERE date >= DATE_SUB(CURDATE(), INTERVAL " . (int)$days . " DAY)
            GROUP BY DATE(date)
            ORDER BY date ASC
        ");
        
        $data = [];
        while ($row = mysql_fetch_array($result)) {
            $data[] = $row;
        }
        
        return $data;
    }
    
    public static function getTopProducts()
    {
        $result = full_query("
            SELECT 
                p.name,
                COUNT(o.id) as orders,
                SUM(o.amount) as revenue
            FROM " . TABLE_PREFIX . "tblproducts p
            JOIN " . TABLE_PREFIX . "tblorders o ON p.id = o.packageid
            WHERE o.date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
            GROUP BY p.id, p.name
            ORDER BY revenue DESC
            LIMIT 10
        ");
        
        $data = [];
        while ($row = mysql_fetch_array($result)) {
            $data[] = $row;
        }
        
        return $data;
    }
    
    public static function getTopClients()
    {
        $result = full_query("
            SELECT 
                c.id,
                c.firstname,
                c.lastname,
                c.companyname,
                SUM(a.amountin) as total_revenue,
                COUNT(DISTINCT o.id) as orders
            FROM " . TABLE_PREFIX . "tblclients c
            JOIN " . TABLE_PREFIX . "tblaccounts a ON c.id = a.userid
            LEFT JOIN " . TABLE_PREFIX . "tblorders o ON c.id = o.userid
            GROUP BY c.id
            ORDER BY total_revenue DESC
            LIMIT 10
        ");
        
        $data = [];
        while ($row = mysql_fetch_array($result)) {
            $data[] = $row;
        }
        
        return $data;
    }
    
    public static function saveMetric($name, $value, $category = 'general')
    {
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$metricsTable . "
            (metric_name, metric_value, category, recorded_at)
            VALUES (
                '" . db_escape_string($name) . "',
                " . (float)$value . ",
                '" . db_escape_string($category) . "',
                NOW()
            )
        ");
    }
    
    public static function getMetricHistory($name, $days = 30)
    {
        $result = full_query("
            SELECT metric_value, recorded_at
            FROM " . TABLE_PREFIX . self::$metricsTable . "
            WHERE metric_name = '" . db_escape_string($name) . "'
            AND recorded_at >= DATE_SUB(NOW(), INTERVAL " . (int)$days . " DAY)
            ORDER BY recorded_at ASC
        ");
        
        $data = [];
        while ($row = mysql_fetch_array($result)) {
            $data[] = $row;
        }
        
        return $data;
    }
}

function kpi_dashboard_activate()
{
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_kpi_metrics (
            id INT AUTO_INCREMENT PRIMARY KEY,
            metric_name VARCHAR(100) NOT NULL,
            metric_value DECIMAL(15,2),
            category VARCHAR(50),
            recorded_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("CREATE INDEX idx_metric_name ON " . TABLE_PREFIX . "mod_kpi_metrics(metric_name)");
    full_query("CREATE INDEX idx_recorded_at ON " . TABLE_PREFIX . "mod_kpi_metrics(recorded_at)");
    
    return ['status' => 'success', 'description' => 'KPI Dashboard activated'];
}

function kpi_dashboard_deactivate()
{
    return ['status' => 'success', 'description' => 'KPI Dashboard deactivated'];
}

function kpi_dashboard_config()
{
    return [
        'name' => 'KPI Dashboard',
        'description' => 'Real-time KPI dashboard with charts and metrics',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'refresh_interval' => [
                'Type' => 'dropdown',
                'FriendlyName' => 'Refresh Interval',
                'Options' => [
                    '30' => '30 seconds',
                    '60' => '1 minute',
                    '300' => '5 minutes',
                    '600' => '10 minutes'
                ],
                'Default' => '60'
            ],
            'chart_days' => [
                'Type' => 'text',
                'FriendlyName' => 'Chart History (Days)',
                'Default' => '30'
            ]
        ]
    ];
}
```