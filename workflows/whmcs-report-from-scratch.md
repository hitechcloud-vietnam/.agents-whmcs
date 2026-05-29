# WHMCS Report Module From Scratch Workflow

## Description
Create a custom report module for WHMCS administrative reporting.

## Prerequisites
- WHMCS 7.0+
- PHP 8.1+
- Admin access

## Steps

### Step 1: Create Report Directory
```bash
mkdir -p /var/www/whmcs/modules/reports/clicodes_custom_report
```

### Step 2: Create Report Module
```php
<?php
/**
 * WHMCS Report Module - CLICodes Custom Report
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Module\Report;

if (!class_exists('WHMCS\Module\Report')) {
    return;
}

/**
 * Report Definition
 */
class CLICodes_Custom_Report extends Report
{
    protected $reportName = 'Custom Business Report';
    
    /**
     * Report metadata
     */
    public function saySummary()
    {
        return 'Custom business metrics and KPIs';
    }
    
    /**
     * Get report results
     */
    public function execute()
    {
        $dateRange = $this->getDateRange();
        $startDate = $dateRange['start'];
        $endDate = $dateRange['end'];
        
        $pdo = Capsule::connection()->getPdo();
        
        // Revenue Data
        $stmt = $pdo->prepare("
            SELECT 
                DATE_FORMAT(date, '%Y-%m') as month,
                SUM(total) as revenue,
                COUNT(*) as invoices
            FROM tblinvoices 
            WHERE status = 'Paid' 
            AND date BETWEEN ? AND ?
            GROUP BY DATE_FORMAT(date, '%Y-%m')
            ORDER BY month DESC
            LIMIT 12
        ");
        $stmt->execute([$startDate, $endDate]);
        $revenueData = $stmt->fetchAll(PDO::FETCH_ASSOC);
        
        // Client Acquisition
        $stmt = $pdo->prepare("
            SELECT 
                DATE_FORMAT(created_at, '%Y-%m') as month,
                COUNT(*) as new_clients
            FROM tblclients
            WHERE created_at BETWEEN ? AND ?
            GROUP BY DATE_FORMAT(created_at, '%Y-%m')
            ORDER BY month DESC
        ");
        $stmt->execute([$startDate, $endDate]);
        $clientData = $stmt->fetchAll(PDO::FETCH_ASSOC);
        
        // Top Products
        $stmt = $pdo->prepare("
            SELECT 
                p.name,
                COUNT(*) as orders
            FROM tblhosting h
            JOIN tblproducts p ON h.packageid = p.id
            WHERE h.regdate BETWEEN ? AND ?
            GROUP BY p.id
            ORDER BY orders DESC
            LIMIT 10
        ");
        $stmt->execute([$startDate, $endDate]);
        $productData = $stmt->fetchAll(PDO::FETCH_ASSOC);
        
        // Churn Analysis
        $stmt = $pdo->prepare("
            SELECT 
                COUNT(*) as terminations
            FROM tblhosting
            WHERE domainstatus = 'Terminated'
            AND termdate BETWEEN ? AND ?
        ");
        $stmt->execute([$startDate, $endDate]);
        $churnData = $stmt->fetch(PDO::FETCH_ASSOC);
        
        // Assign to chart data
        $this->assignChartData('revenue', $revenueData);
        $this->assignChartData('clients', $clientData);
        $this->assign('revenueData', $revenueData);
        $this->assign('clientData', $clientData);
        $this->assign('productData', $productData);
        $this->assign('churnCount', $churnData['terminations'] ?? 0);
        
        return true;
    }
    
    /**
     * Assign chart data
     */
    protected function assignChartData($key, $data)
    {
        $labels = [];
        $values = [];
        
        foreach ($data as $row) {
            if (isset($row['month'])) {
                $labels[] = $row['month'];
            }
            foreach ($row as $field => $value) {
                if ($field !== 'month' && $field !== 'name') {
                    $values[] = $value;
                    break;
                }
            }
        }
        
        $this->assign("{$key}Labels", json_encode($labels));
        $this->assign("{$key}Values", json_encode($values));
    }
    
    /**
     * Get table output
     */
    public function getResults()
    {
        if (!$this->reportExecutionHasOutput()) {
            return '<p class="alert alert-info">No data found for the selected date range.</p>';
        }
        
        $revenueData = $this->get('revenueData', []);
        $productData = $this->get('productData', []);
        $churnCount = $this->get('churnCount', 0);
        
        $html = '<div class="report-container">';
        
        // Summary Cards
        $html .= '<div class="row">';
        $html .= '<div class="col-md-4"><div class="panel panel-default"><div class="panel-body text-center">';
        $html .= '<h3>' . formatCurrency(array_sum(array_column($revenueData, 'revenue'))) . '</h3>';
        $html .= '<p>Total Revenue</p></div></div></div>';
        $html .= '<div class="col-md-4"><div class="panel panel-default"><div class="panel-body text-center">';
        $html .= '<h3>' . count(array_column($revenueData, 'invoices')) . '</h3>';
        $html .= '<p>Invoices Paid</p></div></div></div>';
        $html .= '<div class="col-md-4"><div class="panel panel-default"><div class="panel-body text-center">';
        $html .= '<h3>' . $churnCount . '</h3>';
        $html .= '<p>Terminated Services</p></div></div></div>';
        $html .= '</div>';
        
        // Top Products Table
        $html .= '<table class="table table-striped">';
        $html .= '<thead><tr><th>Product</th><th>Orders</th></tr></thead>';
        $html .= '<tbody>';
        foreach ($productData as $product) {
            $html .= '<tr>';
            $html .= '<td>' . htmlspecialchars($product['name']) . '</td>';
            $html .= '<td>' . $product['orders'] . '</td>';
            $html .= '</tr>';
        }
        $html .= '</tbody></table>';
        
        $html .= '</div>';
        
        return $html;
    }
    
    /**
     * Available tabs
     */
    public function tabs()
    {
        return [];
    }
    
    /**
     * Additional tab output
     */
    public function tabOutput($tab)
    {
        return '';
    }
}
```

### Step 3: Install Report
```bash
# Copy report
cp -r clicodes_custom_report /var/www/whmcs/modules/reports/

# Set permissions
chown -R www-data:www-data /var/www/whmcs/modules/reports/clicodes_custom_report
chmod 644 /var/www/whmcs/modules/reports/clicodes_custom_report/*.php

# Enable in WHMCS Admin
# Go to: Configuration > System Settings > Reports
# Find "Custom Business Report"
# Enable and assign permissions
```

## Tags
- report
- analytics
- module-development
- business-intelligence