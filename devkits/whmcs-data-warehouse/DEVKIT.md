# WHMCS Data Warehouse Module - DEVKIT

## Module Information
- **Name**: Data Warehouse
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: Central data repository for business intelligence and reporting

## Installation
1. Copy to `/modules/addons/data_warehouse/`
2. Activate via WHMCS Admin

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('DailyCronJob', 1, function($vars) {
    DataWarehouse::syncData();
});

add_hook('OrderPlaced', 1, function($vars) {
    DataWarehouse::recordEvent('order', $vars);
});

add_hook('InvoicePaid', 1, function($vars) {
    DataWarehouse::recordEvent('payment', $vars);
});

add_hook('TicketCreated', 1, function($vars) {
    DataWarehouse::recordEvent('ticket', $vars);
});
```

### includes/DataWarehouse.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class DataWarehouse
{
    private static $factTables = [
        'orders' => 'mod_dw_fact_orders',
        'payments' => 'mod_dw_fact_payments',
        'tickets' => 'mod_dw_fact_tickets',
        'services' => 'mod_dw_fact_services'
    ];
    
    private static $dimTables = [
        'clients' => 'mod_dw_dim_clients',
        'products' => 'mod_dw_dim_products',
        'time' => 'mod_dw_dim_time',
        'products' => 'mod_dw_dim_products'
    ];
    
    public static function recordEvent($eventType, $data)
    {
        $factTable = self::$factTables[$eventType] ?? null;
        
        if (!$factTable) {
            return false;
        }
        
        $insertData = self::prepareFactData($eventType, $data);
        
        $columns = implode(', ', array_keys($insertData));
        $values = implode(', ', array_values($insertData));
        
        full_query("
            INSERT INTO " . TABLE_PREFIX . $factTable . " (" . $columns . ")
            VALUES (" . $values . ")
        ");
        
        return true;
    }
    
    private static function prepareFactData($eventType, $data)
    {
        $dateKey = date('Ymd');
        
        switch ($eventType) {
            case 'order':
                return [
                    'order_id' => (int)($data['orderid'] ?? 0),
                    'client_key' => (int)($data['userid'] ?? 0),
                    'product_key' => (int)($data['productid'] ?? 0),
                    'date_key' => $dateKey,
                    'amount' => (float)($data['amount'] ?? 0),
                    'status' => "'" . db_escape_string($data['status'] ?? 'Pending') . "'",
                    'created_at' => 'NOW()'
                ];
                
            case 'payment':
                return [
                    'payment_id' => (int)($data['invoiceid'] ?? 0),
                    'client_key' => (int)($data['userid'] ?? 0),
                    'date_key' => $dateKey,
                    'amount' => (float)($data['amount'] ?? 0),
                    'payment_method' => "'" . db_escape_string($data['paymentmethod'] ?? 'Unknown') . "'",
                    'created_at' => 'NOW()'
                ];
                
            case 'ticket':
                return [
                    'ticket_id' => (int)($data['ticketid'] ?? 0),
                    'client_key' => (int)($data['userid'] ?? 0),
                    'department_key' => (int)($data['deptid'] ?? 0),
                    'date_key' => $dateKey,
                    'priority' => "'" . db_escape_string($data['priority'] ?? 'Medium') . "'",
                    'status' => "'" . db_escape_string($data['status'] ?? 'Open') . "'",
                    'created_at' => 'NOW()'
                ];
                
            case 'service':
                return [
                    'service_id' => (int)($data['serviceid'] ?? 0),
                    'client_key' => (int)($data['userid'] ?? 0),
                    'product_key' => (int)($data['packageid'] ?? 0),
                    'date_key' => $dateKey,
                    'monthly_value' => (float)($data['monthly'] ?? 0),
                    'status' => "'" . db_escape_string($data['domainstatus'] ?? 'Active') . "'",
                    'created_at' => 'NOW()'
                ];
                
            default:
                return [];
        }
    }
    
    public static function syncData()
    {
        self::syncDimension('clients');
        self::syncDimension('products');
        
        self::updateTimeDimension();
        
        return ['status' => 'success', 'synced_at' => date('Y-m-d H:i:s')];
    }
    
    private static function syncDimension($dimType)
    {
        $dimTable = self::$dimTables[$dimType] ?? null;
        
        if (!$dimTable) {
            return false;
        }
        
        switch ($dimType) {
            case 'clients':
                $result = full_query("
                    SELECT id, email, firstname, lastname, companyname, country, created_at
                    FROM " . TABLE_PREFIX . "tblclients
                ");
                
                while ($client = mysql_fetch_array($result)) {
                    full_query("
                        INSERT INTO " . TABLE_PREFIX . $dimTable . " 
                        (dim_key, name, email, company, country, created_at, updated_at)
                        VALUES (
                            " . (int)$client['id'] . ",
                            '" . db_escape_string($client['firstname'] . ' ' . $client['lastname']) . "',
                            '" . db_escape_string($client['email']) . "',
                            '" . db_escape_string($client['companyname'] ?? '') . "',
                            '" . db_escape_string($client['country'] ?? '') . "',
                            '" . db_escape_string($client['created_at']) . "',
                            NOW()
                        )
                        ON DUPLICATE KEY UPDATE 
                            name = VALUES(name),
                            email = VALUES(email),
                            company = VALUES(company),
                            country = VALUES(country),
                            updated_at = NOW()
                    ");
                }
                break;
                
            case 'products':
                $result = full_query("
                    SELECT id, name, type, description
                    FROM " . TABLE_PREFIX . "tblproducts
                ");
                
                while ($product = mysql_fetch_array($result)) {
                    full_query("
                        INSERT INTO " . TABLE_PREFIX . $dimTable . " 
                        (dim_key, name, type, description, updated_at)
                        VALUES (
                            " . (int)$product['id'] . ",
                            '" . db_escape_string($product['name']) . "',
                            '" . db_escape_string($product['type'] ?? 'hosting') . "',
                            '" . db_escape_string(substr($product['description'] ?? '', 0, 500)) . "',
                            NOW()
                        )
                        ON DUPLICATE KEY UPDATE 
                            name = VALUES(name),
                            updated_at = NOW()
                    ");
                }
                break;
        }
        
        return true;
    }
    
    private static function updateTimeDimension()
    {
        $startDate = date('Y-01-01');
        $endDate = date('Y-12-31');
        
        $current = strtotime($startDate);
        $end = strtotime($endDate);
        
        while ($current <= $end) {
            $dateKey = date('Ymd', $current);
            $date = date('Y-m-d', $current);
            
            full_query("
                INSERT INTO " . TABLE_PREFIX . "mod_dw_dim_time 
                (date_key, date, year, quarter, month, day, day_of_week, week_of_year)
                VALUES (
                    " . (int)$dateKey . ",
                    '" . db_escape_string($date) . "',
                    " . (int)date('Y', $current) . ",
                    " . (int)ceil(date('n', $current) / 3) . ",
                    " . (int)date('n', $current) . ",
                    " . (int)date('j', $current) . ",
                    " . (int)date('w', $current) . ",
                    " . (int)date('W', $current) . "
                )
                ON DUPLICATE KEY UPDATE date = VALUES(date)
            ");
            
            $current = strtotime('+1 day', $current);
        }
    }
    
    public static function generateReport($reportType, $params = [])
    {
        switch ($reportType) {
            case 'revenue':
                return self::revenueReport($params);
                
            case 'customer_growth':
                return self::customerGrowthReport($params);
                
            case 'service_usage':
                return self::serviceUsageReport($params);
                
            case 'support_metrics':
                return self::supportMetricsReport($params);
                
            default:
                return [];
        }
    }
    
    private function revenueReport($params)
    {
        $startDate = $params['start_date'] ?? date('Y-01-01');
        $endDate = $params['end_date'] ?? date('Y-m-d');
        
        $result = full_query("
            SELECT 
                t.date_key,
                SUM(f.amount) as total_revenue,
                COUNT(DISTINCT f.client_key) as unique_customers
            FROM " . TABLE_PREFIX . "mod_dw_fact_payments f
            JOIN " . TABLE_PREFIX . "mod_dw_dim_time t ON f.date_key = t.date_key
            WHERE t.date BETWEEN '" . db_escape_string($startDate) . "' AND '" . db_escape_string($endDate) . "'
            GROUP BY t.date_key
            ORDER BY t.date_key ASC
        ");
        
        $data = [];
        while ($row = mysql_fetch_array($result)) {
            $data[] = $row;
        }
        
        return $data;
    }
    
    private function customerGrowthReport($params)
    {
        $result = full_query("
            SELECT 
                t.year,
                t.month,
                COUNT(DISTINCT c.dim_key) as total_customers,
                COUNT(DISTINCT CASE WHEN f.order_id IS NOT NULL THEN c.dim_key END) as active_customers
            FROM " . TABLE_PREFIX . "mod_dw_dim_clients c
            LEFT JOIN " . TABLE_PREFIX . "mod_dw_fact_orders f ON c.dim_key = f.client_key
            JOIN " . TABLE_PREFIX . "mod_dw_dim_time t ON c.created_at LIKE CONCAT(t.date, '%')
            GROUP BY t.year, t.month
            ORDER BY t.year, t.month
        ");
        
        $data = [];
        while ($row = mysql_fetch_array($result)) {
            $data[] = $row;
        }
        
        return $data;
    }
    
    private function serviceUsageReport($params)
    {
        $result = full_query("
            SELECT 
                p.name as product_name,
                COUNT(DISTINCT s.service_id) as total_services,
                SUM(s.monthly_value) as total_mrr,
                COUNT(DISTINCT s.client_key) as unique_customers
            FROM " . TABLE_PREFIX . "mod_dw_fact_services s
            JOIN " . TABLE_PREFIX . "mod_dw_dim_products p ON s.product_key = p.dim_key
            WHERE s.status = 'Active'
            GROUP BY p.name
            ORDER BY total_mrr DESC
        ");
        
        $data = [];
        while ($row = mysql_fetch_array($result)) {
            $data[] = $row;
        }
        
        return $data;
    }
    
    private function supportMetricsReport($params)
    {
        $result = full_query("
            SELECT 
                t.date_key,
                COUNT(*) as total_tickets,
                SUM(CASE WHEN t.status = 'Closed' THEN 1 ELSE 0 END) as resolved,
                SUM(CASE WHEN t.priority = 'High' THEN 1 ELSE 0 END) as high_priority
            FROM " . TABLE_PREFIX . "mod_dw_fact_tickets t
            GROUP BY t.date_key
            ORDER BY t.date_key DESC
            LIMIT 30
        ");
        
        $data = [];
        while ($row = mysql_fetch_array($result)) {
            $data[] = $row;
        }
        
        return $data;
    }
}

function data_warehouse_activate()
{
    // Create fact tables
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_dw_fact_orders (
            id INT AUTO_INCREMENT PRIMARY KEY,
            order_id INT NOT NULL,
            client_key INT NOT NULL,
            product_key INT,
            date_key INT NOT NULL,
            amount DECIMAL(10,2),
            status VARCHAR(50),
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_dw_fact_payments (
            id INT AUTO_INCREMENT PRIMARY KEY,
            payment_id INT NOT NULL,
            client_key INT NOT NULL,
            date_key INT NOT NULL,
            amount DECIMAL(10,2),
            payment_method VARCHAR(50),
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_dw_fact_tickets (
            id INT AUTO_INCREMENT PRIMARY KEY,
            ticket_id INT NOT NULL,
            client_key INT NOT NULL,
            department_key INT,
            date_key INT NOT NULL,
            priority VARCHAR(20),
            status VARCHAR(20),
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_dw_fact_services (
            id INT AUTO_INCREMENT PRIMARY KEY,
            service_id INT NOT NULL,
            client_key INT NOT NULL,
            product_key INT,
            date_key INT NOT NULL,
            monthly_value DECIMAL(10,2),
            status VARCHAR(20),
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    // Create dimension tables
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_dw_dim_clients (
            dim_key INT PRIMARY KEY,
            name VARCHAR(255),
            email VARCHAR(255),
            company VARCHAR(255),
            country VARCHAR(100),
            created_at DATETIME,
            updated_at DATETIME
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_dw_dim_products (
            dim_key INT PRIMARY KEY,
            name VARCHAR(255),
            type VARCHAR(50),
            description TEXT,
            updated_at DATETIME
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_dw_dim_time (
            date_key INT PRIMARY KEY,
            date DATE,
            year INT,
            quarter INT,
            month INT,
            day INT,
            day_of_week INT,
            week_of_year INT
        )
    ");
    
    return ['status' => 'success', 'description' => 'Data Warehouse activated'];
}

function data_warehouse_deactivate()
{
    return ['status' => 'success', 'description' => 'Data Warehouse deactivated'];
}

function data_warehouse_config()
{
    return [
        'name' => 'Data Warehouse',
        'description' => 'Central data repository for business intelligence',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'sync_interval' => [
                'Type' => 'dropdown',
                'FriendlyName' => 'Sync Interval',
                'Options' => [
                    '1' => 'Every Hour',
                    '6' => 'Every 6 Hours',
                    '12' => 'Every 12 Hours',
                    '24' => 'Daily'
                ],
                'Default' => '6'
            ],
            'retention_days' => [
                'Type' => 'text',
                'FriendlyName' => 'Data Retention (Days)',
                'Default' => '365'
            ]
        ]
    ];
}
```