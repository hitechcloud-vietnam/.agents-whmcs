# WHMCS Report Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-report-module/
├── report-module.php        # Main module
├── lib/
│   ├── ReportBuilder.php     # Report builder class
│   ├── DataSources.php       # Data source implementations
│   └── ExportHandler.php     # Export functionality
├── reports/
│   ├── SalesReport.php       # Sales report
│   ├── ClientReport.php      # Client report
│   └── ServiceReport.php     # Service report
└── templates/
    └── report-config.tpl     # Admin configuration
```

## Main Report Module

```php
<?php
/**
 * WHMCS Report Module
 * DevKit Template
 * 
 * Custom report module with data sources and export
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

use WHMCS\Database\Capsule;

/**
 * Config function
 */
function {module}_config(): array {
    return [
        'name' => '{Report Module}',
        'description' => 'Custom reporting with multiple data sources',
        'version' => '1.0',
        'author' => '{Author}',
    ];
}

/**
 * Activate
 */
function {module}_activate(): array {
    Capsule::schema()->create('mod_{module}_reports', function($t) {
        $t->increments('id');
        $t->string('report_name');
        $t->string('report_type');
        $t->text('config');
        $t->boolean('is_scheduled');
        $t->string('schedule'); // cron expression
        $t->string('recipients'); // JSON array
        $t->timestamp('last_run')->nullable();
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_report_data', function($t) {
        $t->increments('id');
        $t->integer('report_id');
        $t->text('data');
        $t->timestamp('generated_at');
    });
    
    Capsule::schema()->create('mod_{module}_scheduled_reports', function($t) {
        $t->increments('id');
        $t->string('report_name');
        $t->string('report_type');
        $t->string('schedule');
        $t->text('config');
        $t->text('recipients');
        $t->string('format'); // pdf, csv, xlsx
        $t->boolean('is_active');
        $t->timestamp('last_run')->nullable();
        $t->timestamp('next_run')->nullable();
    });
    
    // Register default reports
    {module}_registerDefaultReports();
    
    return ['status' => 'success', 'description' => 'Report Module activated'];
}

/**
 * Deactivate
 */
function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_{module}_reports');
    Capsule::schema()->dropIfExists('mod_{module}_report_data');
    Capsule::schema()->dropIfExists('mod_{module}_scheduled_reports');
    
    return ['status' => 'success'];
}

/**
 * Register default reports
 */
function {module}_registerDefaultReports(): void {
    $reports = [
        [
            'name' => 'Monthly Sales Report',
            'type' => 'sales',
            'config' => json_encode([
                'period' => 'month',
                'group_by' => 'day',
                'metrics' => ['revenue', 'orders', 'avg_order_value'],
            ]),
        ],
        [
            'name' => 'Client Activity Report',
            'type' => 'client',
            'config' => json_encode([
                'period' => 'month',
                'metrics' => ['new_clients', 'active_clients', 'churn'],
            ]),
        ],
        [
            'name' => 'Service Usage Report',
            'type' => 'service',
            'config' => json_encode([
                'period' => 'week',
                'metrics' => ['active_services', 'new_signups', 'cancellations'],
            ]),
        ],
    ];
    
    foreach ($reports as $report) {
        Capsule::table('mod_{module}_reports')->insert([
            'report_name' => $report['name'],
            'report_type' => $report['type'],
            'config' => $report['config'],
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

/**
 * Output function (Admin Interface)
 */
function {module}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'dashboard';
    $reportType = $_REQUEST['type'] ?? 'sales';
    
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
    }
    
    switch ($action) {
        case 'view':
            {module}_viewReport($reportType);
            break;
        case 'generate':
            {module}_generateReport();
            break;
        case 'schedule':
            {module}_manageSchedule();
            break;
        case 'export':
            {module}_exportReport();
            break;
        default:
            {module}_showDashboard();
    }
}
```

## Report Builder

```php
<?php
/**
 * Report Builder
 * Creates and formats reports
 */

namespace ReportModule;

use WHMCS\Database\Capsule;

class ReportBuilder {
    
    private string $reportType;
    private array $config = [];
    private array $data = [];
    
    /**
     * Create new report
     */
    public function __construct(string $reportType, array $config = []) {
        $this->reportType = $reportType;
        $this->config = $config;
    }
    
    /**
     * Set report configuration
     */
    public function setConfig(array $config): self {
        $this->config = array_merge($this->config, $config);
        return $this;
    }
    
    /**
     * Generate report data
     */
    public function generate(): array {
        $method = 'generate' . ucfirst($this->reportType) . 'Report';
        
        if (method_exists($this, $method)) {
            $this->data = $this->$method();
        } else {
            $this->data = $this->generateGenericReport();
        }
        
        return $this->data;
    }
    
    /**
     * Generate sales report
     */
    private function generateSalesReport(): array {
        $period = $this->config['period'] ?? 'month';
        $groupBy = $this->config['group_by'] ?? 'day';
        $metrics = $this->config['metrics'] ?? ['revenue'];
        
        $data = [];
        
        // Determine date range
        $startDate = date('Y-m-01');
        $endDate = date('Y-m-t');
        
        if ($period === 'week') {
            $startDate = date('Y-m-d', strtotime('-7 days'));
            $endDate = date('Y-m-d');
        } elseif ($period === 'year') {
            $startDate = date('Y-01-01');
            $endDate = date('Y-12-31');
        }
        
        // Revenue data
        if (in_array('revenue', $metrics)) {
            $data['revenue'] = $this->getRevenueData($startDate, $endDate, $groupBy);
        }
        
        // Orders data
        if (in_array('orders', $metrics)) {
            $data['orders'] = $this->getOrdersData($startDate, $endDate, $groupBy);
        }
        
        // Average order value
        if (in_array('avg_order_value', $metrics)) {
            $data['avg_order_value'] = $this->getAverageOrderValue($startDate, $endDate);
        }
        
        // Top products
        $data['top_products'] = $this->getTopProducts($startDate, $endDate);
        
        // Summary
        $data['summary'] = [
            'total_revenue' => array_sum(array_column($data['revenue'] ?? [], 'value')),
            'total_orders' => array_sum(array_column($data['orders'] ?? [], 'value')),
            'period' => $period,
            'start_date' => $startDate,
            'end_date' => $endDate,
        ];
        
        return $data;
    }
    
    /**
     * Generate client report
     */
    private function generateClientReport(): array {
        $period = $this->config['period'] ?? 'month';
        
        // Date range
        $startDate = date('Y-m-01');
        $endDate = date('Y-m-t');
        
        if ($period === 'week') {
            $startDate = date('Y-m-d', strtotime('-7 days'));
            $endDate = date('Y-m-d');
        }
        
        // New clients
        $newClients = Capsule::table('tblclients')
            ->whereBetween('created_at', [$startDate, $endDate . ' 23:59:59'])
            ->count();
        
        // Active clients
        $activeClients = Capsule::table('tblclients')
            ->where('status', 'Active')
            ->count();
        
        // Clients by country
        $byCountry = Capsule::table('tblclients')
            ->selectRaw('country, COUNT(*) as count')
            ->groupBy('country')
            ->orderBy('count', 'desc')
            ->limit(10)
            ->get();
        
        // Clients by signup source
        $bySource = Capsule::table('tblclients')
            ->selectRaw('COALESCE(affiliateid, 0) as source, COUNT(*) as count')
            ->whereBetween('created_at', [$startDate, $endDate . ' 23:59:59'])
            ->groupBy('affiliateid')
            ->get();
        
        return [
            'new_clients' => $newClients,
            'active_clients' => $activeClients,
            'total_clients' => Capsule::table('tblclients')->count(),
            'by_country' => $byCountry->toArray(),
            'by_source' => $bySource->toArray(),
            'period' => $period,
        ];
    }
    
    /**
     * Generate service report
     */
    private function generateServiceReport(): array {
        $period = $this->config['period'] ?? 'week';
        
        $startDate = date('Y-m-d', strtotime('-7 days'));
        $endDate = date('Y-m-d');
        
        // Active services
        $activeServices = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->count();
        
        // New services this period
        $newServices = Capsule::table('tblhosting')
            ->whereBetween('regdate', [$startDate, $endDate . ' 23:59:59'])
            ->count();
        
        // Terminated services
        $terminated = Capsule::table('tblhosting')
            ->where('domainstatus', 'Terminated')
            ->whereBetween('modified', [$startDate, $endDate . ' 23:59:59'])
            ->count();
        
        // By product
        $byProduct = Capsule::table('tblhosting')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->selectRaw('tblproducts.name, COUNT(*) as count')
            ->where('tblhosting.domainstatus', 'Active')
            ->groupBy('tblproducts.name')
            ->get();
        
        return [
            'active_services' => $activeServices,
            'new_services' => $newServices,
            'terminated' => $terminated,
            'by_product' => $byProduct->toArray(),
            'retention_rate' => $this->calculateRetentionRate(),
        ];
    }
    
    /**
     * Generic report generator
     */
    private function generateGenericReport(): array {
        return [
            'generated_at' => date('Y-m-d H:i:s'),
            'report_type' => $this->reportType,
            'config' => $this->config,
        ];
    }
    
    /**
     * Get revenue data
     */
    private function getRevenueData(string $start, string $end, string $groupBy): array {
        $group = match ($groupBy) {
            'day' => "DATE(datepaid)",
            'week' => "YEARWEEK(datepaid)",
            'month' => "DATE_FORMAT(datepaid, '%Y-%m')",
            default => "DATE(datepaid)",
        };
        
        return Capsule::table('tblinvoices')
            ->selectRaw("{$group} as period, SUM(total) as value, COUNT(*) as count")
            ->where('status', 'Paid')
            ->whereBetween('datepaid', [$start, $end . ' 23:59:59'])
            ->groupByRaw($group)
            ->orderBy('period')
            ->get()
            ->map(function($row) {
                return [
                    'date' => $row->period,
                    'value' => (float) $row->value,
                    'count' => (int) $row->count,
                ];
            })
            ->toArray();
    }
    
    /**
     * Get orders data
     */
    private function getOrdersData(string $start, string $end, string $groupBy): array {
        $group = match ($groupBy) {
            'day' => "DATE(date)",
            'week' => "YEARWEEK(date)",
            'month' => "DATE_FORMAT(date, '%Y-%m')",
            default => "DATE(date)",
        };
        
        return Capsule::table('tblorders')
            ->selectRaw("{$group} as period, COUNT(*) as value")
            ->whereBetween('date', [$start, $end . ' 23:59:59'])
            ->groupByRaw($group)
            ->orderBy('period')
            ->get()
            ->map(function($row) {
                return [
                    'date' => $row->period,
                    'value' => (int) $row->value,
                ];
            })
            ->toArray();
    }
    
    /**
     * Get average order value
     */
    private function getAverageOrderValue(string $start, string $end): float {
        $result = Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->whereBetween('datepaid', [$start, $end . ' 23:59:59'])
            ->selectRaw('AVG(total) as avg')
            ->first();
        
        return (float) ($result->avg ?? 0);
    }
    
    /**
     * Get top products
     */
    private function getTopProducts(string $start, string $end): array {
        return Capsule::table('tblorders')
            ->join('tblproducts', 'tblorders.productid', '=', 'tblproducts.id')
            ->selectRaw('tblproducts.name, COUNT(*) as count, SUM(tblorders.amount) as revenue')
            ->whereBetween('tblorders.date', [$start, $end . ' 23:59:59'])
            ->groupBy('tblproducts.name')
            ->orderBy('count', 'desc')
            ->limit(10)
            ->get()
            ->toArray();
    }
    
    /**
     * Calculate retention rate
     */
    private function calculateRetentionRate(): float {
        $totalStart = Capsule::table('tblhosting')
            ->whereDate('regdate', '<', date('Y-m-d', strtotime('-30 days')))
            ->count();
        
        $stillActive = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->whereDate('regdate', '<', date('Y-m-d', strtotime('-30 days')))
            ->count();
        
        if ($totalStart === 0) {
            return 100.0;
        }
        
        return round(($stillActive / $totalStart) * 100, 2);
    }
    
    /**
     * Export report as array
     */
    public function toArray(): array {
        return [
            'type' => $this->reportType,
            'generated_at' => date('Y-m-d H:i:s'),
            'config' => $this->config,
            'data' => $this->data,
        ];
    }
    
    /**
     * Export as CSV
     */
    public function toCsv(): string {
        $csv = [];
        
        foreach ($this->data as $section => $rows) {
            if (is_array($rows)) {
                $csv[] = "# {$section}";
                
                if (isset($rows[0]) && is_array($rows[0])) {
                    // Header row
                    $csv[] = implode(',', array_keys((array) $rows[0]));
                    
                    foreach ($rows as $row) {
                        $csv[] = implode(',', array_values((array) $row));
                    }
                }
                
                $csv[] = ''; // Empty line between sections
            }
        }
        
        return implode("\n", $csv);
    }
    
    /**
     * Export as JSON
     */
    public function toJson(): string {
        return json_encode($this->toArray(), JSON_PRETTY_PRINT);
    }
}
```

## Data Sources

```php
<?php
/**
 * Data Source Implementations
 */

namespace ReportModule;

use WHMCS\Database\Capsule;

class DataSources {
    
    /**
     * Get invoice data source
     */
    public static function invoices(array $filters = []): array {
        $query = Capsule::table('tblinvoices');
        
        if (!empty($filters['status'])) {
            $query->where('status', $filters['status']);
        }
        
        if (!empty($filters['client_id'])) {
            $query->where('userid', $filters['client_id']);
        }
        
        if (!empty($filters['start_date'])) {
            $query->where('date', '>=', $filters['start_date']);
        }
        
        if (!empty($filters['end_date'])) {
            $query->where('date', '<=', $filters['end_date']);
        }
        
        return $query->get()->toArray();
    }
    
    /**
     * Get client data source
     */
    public static function clients(array $filters = []): array {
        $query = Capsule::table('tblclients')
            ->join('tblorders', 'tblclients.id', '=', 'tblorders.userid')
            ->join('tblhosting', 'tblclients.id', '=', 'tblhosting.userid')
            ->selectRaw('tblclients.*, COUNT(DISTINCT tblorders.id) as order_count, COUNT(DISTINCT tblhosting.id) as service_count')
            ->groupBy('tblclients.id');
        
        if (!empty($filters['status'])) {
            $query->where('tblclients.status', $filters['status']);
        }
        
        if (!empty($filters['country'])) {
            $query->where('tblclients.country', $filters['country']);
        }
        
        return $query->get()->toArray();
    }
    
    /**
     * Get service data source
     */
    public static function services(array $filters = []): array {
        $query = Capsule::table('tblhosting')
            ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->selectRaw('tblhosting.*, tblclients.firstname, tblclients.lastname, tblclients.email, tblproducts.name as product_name');
        
        if (!empty($filters['status'])) {
            $query->where('tblhosting.domainstatus', $filters['status']);
        }
        
        if (!empty($filters['product_id'])) {
            $query->where('tblhosting.packageid', $filters['product_id']);
        }
        
        return $query->get()->toArray();
    }
    
    /**
     * Get domain data source
     */
    public static function domains(array $filters = []): array {
        $query = Capsule::table('tbldomains')
            ->join('tblclients', 'tbldomains.userid', '=', 'tblclients.id')
            ->selectRaw('tbldomains.*, tblclients.firstname, tblclients.lastname, tblclients.email');
        
        if (!empty($filters['status'])) {
            $query->where('tbldomains.status', $filters['status']);
        }
        
        if (!empty($filters['expiring_days'])) {
            $query->whereRaw('DATEDIFF(tbldomains.expirydate, CURDATE()) <= ?', [$filters['expiring_days']]);
        }
        
        return $query->get()->toArray();
    }
    
    /**
     * Get ticket data source
     */
    public static function tickets(array $filters = []): array {
        $query = Capsule::table('tbltickets')
            ->join('tblclients', 'tbltickets.userid', '=', 'tblclients.id')
            ->selectRaw('tbltickets.*, tblclients.firstname, tblclients.lastname, tblclients.email');
        
        if (!empty($filters['status'])) {
            $query->where('tbltickets.status', $filters['status']);
        }
        
        if (!empty($filters['department_id'])) {
            $query->where('tbltickets.departmentid', $filters['department_id']);
        }
        
        return $query->get()->toArray();
    }
    
    /**
     * Get transaction data source
     */
    public static function transactions(array $filters = []): array {
        $query = Capsule::table('tblaccounts');
        
        if (!empty($filters['start_date'])) {
            $query->where('date', '>=', $filters['start_date']);
        }
        
        if (!empty($filters['end_date'])) {
            $query->where('date', '<=', $filters['end_date']);
        }
        
        return $query->get()->toArray();
    }
    
    /**
     * Get order data source
     */
    public static function orders(array $filters = []): array {
        $query = Capsule::table('tblorders')
            ->join('tblclients', 'tblorders.userid', '=', 'tblclients.id')
            ->join('tblproducts', 'tblorders.productid', '=', 'tblproducts.id')
            ->selectRaw('tblorders.*, tblclients.firstname, tblclients.lastname, tblclients.email, tblproducts.name as product_name');
        
        if (!empty($filters['status'])) {
            $query->where('tblorders.status', $filters['status']);
        }
        
        if (!empty($filters['date_from'])) {
            $query->where('tblorders.date', '>=', $filters['date_from']);
        }
        
        if (!empty($filters['date_to'])) {
            $query->where('tblorders.date', '<=', $filters['date_to']);
        }
        
        return $query->get()->toArray();
    }
}
```

## Export Handler

```php
<?php
/**
 * Export Handler
 */

namespace ReportModule;

class ExportHandler {
    
    /**
     * Export report as PDF
     */
    public static function toPdf(array $data, string $title): string {
        // In production, use a PDF library like TCPDF or DomPDF
        // This is a simplified example
        
        $html = '<html><head><title>' . htmlspecialchars($title) . '</title>';
        $html .= '<style>body{font-family:Arial;padding:20px;} table{border-collapse:collapse;width:100%;} th,td{border:1px solid #ddd;padding:8px;text-align:left;} th{background:#f5f5f5;} h1{color:#333;} .summary{background:#f9f9f9;padding:15px;margin:20px 0;}</style>';
        $html .= '</head><body>';
        $html .= '<h1>' . htmlspecialchars($title) . '</h1>';
        $html .= '<p>Generated: ' . date('Y-m-d H:i:s') . '</p>';
        
        foreach ($data as $section => $content) {
            $html .= '<h2>' . htmlspecialchars(ucfirst(str_replace('_', ' ', $section))) . '</h2>';
            
            if (is_array($content) && isset($content[0])) {
                $html .= '<table><thead><tr>';
                foreach ((array) $content[0] as $key => $value) {
                    $html .= '<th>' . htmlspecialchars(ucfirst(str_replace('_', ' ', $key))) . '</th>';
                }
                $html .= '</tr></thead><tbody>';
                
                foreach ($content as $row) {
                    $html .= '<tr>';
                    foreach ((array) $row as $value) {
                        $html .= '<td>' . htmlspecialchars((string) $value) . '</td>';
                    }
                    $html .= '</tr>';
                }
                
                $html .= '</tbody></table>';
            } elseif (is_array($content)) {
                $html .= '<div class="summary"><p><strong>Total:</strong> ' . htmlspecialchars((string) $content) . '</p></div>';
            }
        }
        
        $html .= '</body></html>';
        
        return $html;
    }
    
    /**
     * Export report as CSV
     */
    public static function toCsv(array $data): string {
        $output = [];
        
        foreach ($data as $section => $content) {
            if (is_array($content) && isset($content[0])) {
                // Section header
                $output[] = "# " . ucfirst(str_replace('_', ' ', $section));
                
                // Headers
                $headers = array_keys((array) $content[0]);
                $output[] = implode(',', $headers);
                
                // Data rows
                foreach ($content as $row) {
                    $values = array_map(function($v) {
                        return '"' . str_replace('"', '""', $v) . '"';
                    }, array_values((array) $row));
                    $output[] = implode(',', $values);
                }
                
                $output[] = ''; // Empty line
            }
        }
        
        return implode("\n", $output);
    }
    
    /**
     * Export report as Excel (XLSX)
     */
    public static function toExcel(array $data, string $title): string {
        // In production, use PhpSpreadsheet library
        // This returns a simplified representation
        
        return json_encode([
            'title' => $title,
            'generated' => date('Y-m-d H:i:s'),
            'sheets' => $data,
        ]);
    }
    
    /**
     * Export report as HTML
     */
    public static function toHtml(array $data, string $title): string {
        $html = '<!DOCTYPE html>';
        $html .= '<html><head>';
        $html .= '<meta charset="utf-8">';
        $html .= '<title>' . htmlspecialchars($title) . '</title>';
        $html .= '<style>
            body{font-family:Arial,sans-serif;padding:20px;color:#333;}
            h1{color:#222;border-bottom:2px solid #007bff;padding-bottom:10px;}
            h2{color:#555;margin-top:30px;}
            table{border-collapse:collapse;width:100%;margin:15px 0;}
            th,td{border:1px solid #ddd;padding:10px;text-align:left;}
            th{background:#f8f9fa;font-weight:bold;}
            tr:nth-child(even){background:#fafafa;}
            .summary{background:#e9ecef;padding:15px;border-radius:5px;margin:15px 0;}
            .footer{margin-top:30px;padding-top:20px;border-top:1px solid #ddd;color:#666;font-size:12px;}
        </style>';
        $html .= '</head><body>';
        $html .= '<h1>' . htmlspecialchars($title) . '</h1>';
        $html .= '<p><strong>Generated:</strong> ' . date('Y-m-d H:i:s') . '</p>';
        
        foreach ($data as $section => $content) {
            $html .= '<h2>' . htmlspecialchars(ucfirst(str_replace('_', ' ', $section))) . '</h2>';
            
            if (is_array($content) && isset($content[0])) {
                $html .= '<table><thead><tr>';
                foreach ((array) $content[0] as $key => $value) {
                    $html .= '<th>' . htmlspecialchars(ucfirst(str_replace('_', ' ', $key))) . '</th>';
                }
                $html .= '</tr></thead><tbody>';
                
                foreach ($content as $row) {
                    $html .= '<tr>';
                    foreach ((array) $row as $value) {
                        $html .= '<td>' . htmlspecialchars((string) $value) . '</td>';
                    }
                    $html .= '</tr>';
                }
                
                $html .= '</tbody></table>';
            } elseif (is_array($content)) {
                $html .= '<div class="summary"><p><strong>Value:</strong> ' . htmlspecialchars((string) $content) . '</p></div>';
            }
        }
        
        $html .= '<div class="footer">';
        $html .= '<p>Generated by WHMCS Report Module</p>';
        $html .= '</div></body></html>';
        
        return $html;
    }
}
```

## Admin Report View Template

```smarty
<div class="report-module">
    <div class="row">
        <div class="col-md-12">
            <h2>Reports</h2>
            
            <div class="btn-group mb-3">
                <a href="?module={module}&action=view&type=sales" class="btn btn-primary {if $type eq 'sales'}active{/if}">
                    <i class="fa fa-chart-line"></i> Sales
                </a>
                <a href="?module={module}&action=view&type=client" class="btn btn-primary {if $type eq 'client'}active{/if}">
                    <i class="fa fa-users"></i> Clients
                </a>
                <a href="?module={module}&action=view&type=service" class="btn btn-primary {if $type eq 'service'}active{/if}">
                    <i class="fa fa-server"></i> Services
                </a>
            </div>
        </div>
    </div>
    
    <form method="post" action="?module={module}&action=generate">
        <input type="hidden" name="csrf_token" value="{$csrf_token}">
        <input type="hidden" name="report_type" value="{$type}">
        
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">Report Parameters</h3>
            </div>
            <div class="panel-body">
                <div class="row">
                    <div class="col-md-4">
                        <div class="form-group">
                            <label>Period</label>
                            <select name="period" class="form-control">
                                <option value="week">Last 7 Days</option>
                                <option value="month" selected>Last Month</option>
                                <option value="quarter">Last Quarter</option>
                                <option value="year">Last Year</option>
                            </select>
                        </div>
                    </div>
                    <div class="col-md-4">
                        <div class="form-group">
                            <label>Group By</label>
                            <select name="group_by" class="form-control">
                                <option value="day">Day</option>
                                <option value="week">Week</option>
                                <option value="month">Month</option>
                            </select>
                        </div>
                    </div>
                    <div class="col-md-4">
                        <div class="form-group">
                            <label>Format</label>
                            <select name="format" class="form-control">
                                <option value="html">HTML View</option>
                                <option value="csv">CSV Export</option>
                                <option value="pdf">PDF Export</option>
                                <option value="xlsx">Excel Export</option>
                            </select>
                        </div>
                    </div>
                </div>
                
                <button type="submit" class="btn btn-success">
                    <i class="fa fa-chart-bar"></i> Generate Report
                </button>
            </div>
        </div>
    </form>
    
    {if $report_data}
    <div class="panel panel-default">
        <div class="panel-heading">
            <h3 class="panel-title">Report Results: {$report_title}</h3>
            <div class="pull-right">
                <a href="?module={module}&action=export&format=csv&type={$type}" class="btn btn-xs btn-default">
                    <i class="fa fa-download"></i> CSV
                </a>
                <a href="?module={module}&action=export&format=pdf&type={$type}" class="btn btn-xs btn-default">
                    <i class="fa fa-file-pdf"></i> PDF
                </a>
            </div>
        </div>
        <div class="panel-body">
            <pre>{$report_data|json_encode:true}</pre>
        </div>
    </div>
    {/if}
</div>
```

## Checklist

```
Pre-Dev:
□ Define report types
□ Plan data sources
□ Design report layouts
□ Plan export formats

Development:
□ Create main module with tables
□ Implement ReportBuilder class
□ Create data source implementations
□ Implement ExportHandler class
□ Build report templates
□ Add scheduled report support
□ Create admin UI
□ Add export functionality

Testing:
□ Test report generation
□ Verify data accuracy
□ Test export formats
□ Test scheduled reports
□ Verify styling
```