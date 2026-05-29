# WHMCS Report Module Master

## Overview
Master skill for WHMCS custom report development. Covers report module structure, data querying, chart generation, and export functionality.

## Report Module Structure

```php
<?php
// /modules/reports/YourReport.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Carbon;
use WHMCS\Report\AbstractReport;
use WHMCS\Report\HtmlAttributes;
use WHMCS\Report\PageBasedReport;

class YourReport extends AbstractReport
{
    protected $table = 'YourReport';

    // Report metadata
    public $title = 'Your Custom Report';
    public $description = 'Description of what this report displays';

    // Column headers
    protected $headers = [
        'id' => 'ID',
        'client_name' => 'Client Name',
        'email' => 'Email',
        'item_name' => 'Item',
        'amount' => 'Amount',
        'status' => 'Status',
        'created_at' => 'Created',
    ];

    // Default sort
    protected $sortOrder = 'id';
    protected $sortDirection = 'desc';

    // CSV export settings
    protected $csvFileName = 'your_report';

    public function __construct()
    {
        parent::__construct();
    }

    public function getFilterPresets()
    {
        return [
            'today' => [
                'name' => 'Today',
                'from' => Carbon::today()->toDateString(),
                'to' => Carbon::today()->toDateString(),
            ],
            'yesterday' => [
                'name' => 'Yesterday',
                'from' => Carbon::yesterday()->toDateString(),
                'to' => Carbon::yesterday()->toDateString(),
            ],
            'this_month' => [
                'name' => 'This Month',
                'from' => Carbon::now()->startOfMonth()->toDateString(),
                'to' => Carbon::now()->endOfMonth()->toDateString(),
            ],
            'last_month' => [
                'name' => 'Last Month',
                'from' => Carbon::now()->subMonth()->startOfMonth()->toDateString(),
                'to' => Carbon::now()->subMonth()->endOfMonth()->toDateString(),
            ],
            'this_year' => [
                'name' => 'This Year',
                'from' => Carbon::now()->startOfYear()->toDateString(),
                'to' => Carbon::now()->endOfYear()->toDateString(),
            ],
        ];
    }

    public function getTable($results)
    {
        $tableBody = '';

        foreach ($results as $result) {
            $tableBody .= '<tr>';
            $tableBody .= '<td>' . $result['id'] . '</td>';
            $tableBody .= '<td>' . htmlspecialchars($result['client_name']) . '</td>';
            $tableBody .= '<td>' . htmlspecialchars($result['email']) . '</td>';
            $tableBody .= '<td>' . htmlspecialchars($result['item_name']) . '</td>';
            $tableBody .= '<td class="text-right">$' . number_format($result['amount'], 2) . '</td>';
            $tableBody .= '<td><span class="label label-' . $this->getStatusClass($result['status']) . '">' . $result['status'] . '</span></td>';
            $tableBody .= '<td>' . $result['created_at'] . '</td>';
            $tableBody .= '</tr>';
        }

        $table = '<div class="table-responsive">
            <table class="table table-striped table-bordered">
                <thead>
                    <tr>
                        <th>ID</th>
                        <th>Client Name</th>
                        <th>Email</th>
                        <th>Item</th>
                        <th class="text-right">Amount</th>
                        <th>Status</th>
                        <th>Created</th>
                    </tr>
                </thead>
                <tbody>
                    ' . $tableBody . '
                </tbody>
            </table>
        </div>';

        return $table;
    }

    protected function getStatusClass(string $status): string
    {
        $classes = [
            'Active' => 'success',
            'Pending' => 'warning',
            'Suspended' => 'danger',
            'Cancelled' => 'default',
            'Terminated' => 'default',
        ];

        return $classes[$status] ?? 'default';
    }

    public function setTableSelection($post)
    {
        // Set filters from POST data
        $this->filterDateFrom = Carbon::safeCreateDateTime($post['date_from'] ?? null);
        $this->filterDateTo = Carbon::safeCreateDateTime($post['date_to'] ?? null);
        $this->filterStatus = $post['status'] ?? [];
    }

    public function getWhere()
    {
        $where = [];

        // Date filter
        if ($this->filterDateFrom) {
            $where[] = "created_at >= '" . $this->filterDateFrom->toDateTimeString() . "'";
        }

        if ($this->filterDateTo) {
            $where[] = "created_at <= '" . $this->filterDateTo->toDateTimeString() . "'";
        }

        // Status filter
        if (!empty($this->filterStatus)) {
            $statuses = array_map(function($status) {
                return "'" . db_escape_string($status) . "'";
            }, $this->filterStatus);

            $where[] = "status IN (" . implode(", ", $statuses) . ")";
        }

        return implode(" AND ", $where);
    }

    public function getResults()
    {
        $query = $this->getQuery();

        $results = $query->get();

        return $results;
    }

    protected function getQuery()
    {
        $query = \Illuminate\Database\Capsule\Manager::table('your_table')
            ->join('tblclients', 'your_table.client_id', '=', 'tblclients.id')
            ->join('tblhosting', 'your_table.service_id', '=', 'tblhosting.id')
            ->select([
                'your_table.id',
                \Illuminate\Database\Capsule\Manager::raw("CONCAT(tblclients.firstname, ' ', tblclients.lastname) AS client_name"),
                'tblclients.email',
                'tblhosting.domain AS item_name',
                'your_table.amount',
                'your_table.status',
                'your_table.created_at',
            ]);

        // Apply filters
        $where = $this->getWhere();
        if (!empty($where)) {
            $query->whereRaw($where);
        }

        // Apply sorting
        $query->orderBy($this->sortOrder, $this->sortDirection);

        return $query;
    }

    public function getChartData()
    {
        // Get aggregated data for charts
        $monthlyData = \Illuminate\Database\Capsule\Manager::table('your_table')
            ->select(
                \Illuminate\Database\Capsule\Manager::raw("DATE_FORMAT(created_at, '%Y-%m') AS month"),
                \Illuminate\Database\Capsule\Manager::raw("COUNT(*) AS count"),
                \Illuminate\Database\Capsule\Manager::raw("SUM(amount) AS total")
            )
            ->where('created_at', '>=', Carbon::now()->subMonths(12))
            ->groupBy('month')
            ->orderBy('month')
            ->get();

        $labels = [];
        $values = [];

        foreach ($monthlyData as $row) {
            $labels[] = $row->month;
            $values[] = $row->total;
        }

        return [
            'labels' => $labels,
            'datasets' => [
                [
                    'label' => 'Monthly Totals',
                    'data' => $values,
                    'backgroundColor' => 'rgba(52, 152, 219, 0.2)',
                    'borderColor' => 'rgba(52, 152, 219, 1)',
                    'borderWidth' => 2,
                    'fill' => true,
                ],
            ],
        ];
    }

    public function getSummary()
    {
        $stats = \Illuminate\Database\Capsule\Manager::table('your_table')
            ->selectRaw("
                COUNT(*) as total_count,
                SUM(amount) as total_amount,
                AVG(amount) as avg_amount,
                COUNT(DISTINCT client_id) as unique_clients
            ")
            ->first();

        return [
            'total_records' => number_format($stats->total_count),
            'total_amount' => '$' . number_format($stats->total_amount ?? 0, 2),
            'average_amount' => '$' . number_format($stats->avg_amount ?? 0, 2),
            'unique_clients' => number_format($stats->unique_clients),
        ];
    }
}
```

## Advanced Report with Multiple Tabs

```php
<?php
// /modules/reports/DetailedReport.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Carbon;
use WHMCS\Report\AbstractReport;

class DetailedReport extends AbstractReport
{
    protected $table = 'DetailedReport';
    public $title = 'Detailed Analytics Report';
    public $description = 'Comprehensive analytics with multiple views';

    public function __construct()
    {
        parent::__construct();

        // Set default tab
        $this->activeTab = $_GET['tab'] ?? 'overview';
    }

    public function getTabs()
    {
        return [
            'overview' => [
                'label' => 'Overview',
                'icon' => 'fa-dashboard',
            ],
            'clients' => [
                'label' => 'Clients',
                'icon' => 'fa-users',
            ],
            'products' => [
                'label' => 'Products',
                'icon' => 'fa-cube',
            ],
            'revenue' => [
                'label' => 'Revenue',
                'icon' => 'fa-dollar',
            ],
            'export' => [
                'label' => 'Export',
                'icon' => 'fa-download',
            ],
        ];
    }

    public function getResults()
    {
        switch ($this->activeTab) {
            case 'overview':
                return $this->getOverviewData();
            case 'clients':
                return $this->getClientData();
            case 'products':
                return $this->getProductData();
            case 'revenue':
                return $this->getRevenueData();
            default:
                return [];
        }
    }

    protected function getOverviewData(): array
    {
        return [
            'stats' => $this->getSummaryStats(),
            'chart' => $this->getOverviewChart(),
            'recent_activity' => $this->getRecentActivity(),
        ];
    }

    protected function getClientData(): array
    {
        $clients = \Illuminate\Database\Capsule\Manager::table('tblclients')
            ->selectRaw("
                tblclients.*,
                COUNT(tblhosting.id) as service_count,
                SUM(tblinvoices.total) as total_spent
            ")
            ->leftJoin('tblhosting', 'tblclients.id', '=', 'tblhosting.userid')
            ->leftJoin('tblinvoices', function($join) {
                $join->on('tblclients.id', '=', 'tblinvoices.userid')
                    ->where('tblinvoices.status', '=', 'Paid');
            })
            ->groupBy('tblclients.id')
            ->orderBy('total_spent', 'desc')
            ->limit(100)
            ->get();

        return [
            'table' => $this->generateClientTable($clients),
            'total_clients' => count($clients),
        ];
    }

    protected function generateClientTable(array $clients): string
    {
        $html = '<table class="table table-striped">
            <thead>
                <tr>
                    <th>ID</th>
                    <th>Name</th>
                    <th>Email</th>
                    <th>Services</th>
                    <th class="text-right">Total Spent</th>
                    <th>Joined</th>
                </tr>
            </thead>
            <tbody>';

        foreach ($clients as $client) {
            $html .= '<tr>
                <td>#' . $client->id . '</td>
                <td>' . htmlspecialchars($client->firstname . ' ' . $client->lastname) . '</td>
                <td>' . htmlspecialchars($client->email) . '</td>
                <td>' . $client->service_count . '</td>
                <td class="text-right">$' . number_format($client->total_spent ?? 0, 2) . '</td>
                <td>' . date('Y-m-d', strtotime($client->datecreated)) . '</td>
            </tr>';
        }

        $html .= '</tbody></table>';

        return $html;
    }

    protected function getSummaryStats(): array
    {
        $stats = \Illuminate\Database\Capsule\Manager::table('tblclients')
            ->selectRaw("
                (SELECT COUNT(*) FROM tblclients) as total_clients,
                (SELECT COUNT(*) FROM tblhosting WHERE domainstatus = 'Active') as active_services,
                (SELECT COUNT(*) FROM tblorders WHERE status = 'Pending') as pending_orders,
                (SELECT SUM(total) FROM tblinvoices WHERE status = 'Paid' AND date >= CURDATE() - INTERVAL 30 DAY) as monthly_revenue
            ")
            ->first();

        return [
            'total_clients' => number_format($stats->total_clients),
            'active_services' => number_format($stats->active_services),
            'pending_orders' => number_format($stats->pending_orders),
            'monthly_revenue' => '$' . number_format($stats->monthly_revenue ?? 0, 2),
        ];
    }

    protected function getOverviewChart(): array
    {
        $monthlyOrders = \Illuminate\Database\Capsule\Manager::table('tblorders')
            ->selectRaw("
                DATE_FORMAT(date, '%Y-%m') as month,
                COUNT(*) as count,
                SUM(total) as revenue
            ")
            ->where('date', '>=', Carbon::now()->subMonths(12))
            ->groupBy('month')
            ->orderBy('month')
            ->get();

        return [
            'labels' => array_column($monthlyOrders, 'month'),
            'orders' => array_column($monthlyOrders, 'count'),
            'revenue' => array_column($monthlyOrders, 'revenue'),
        ];
    }

    protected function getRecentActivity(): array
    {
        return \Illuminate\Database\Capsule\Manager::table('tblactivitylog')
            ->join('tblusers', 'tblactivitylog.userid', '=', 'tblusers.id')
            ->select('tblactivitylog.*', 'tblusers.username')
            ->orderBy('tblactivitylog.id', 'desc')
            ->limit(10)
            ->get();
    }

    public function exportData(string $format = 'csv')
    {
        $data = $this->getClientData()['clients'] ?? [];

        switch ($format) {
            case 'csv':
                return $this->exportCsv($data);
            case 'xlsx':
                return $this->exportXlsx($data);
            case 'pdf':
                return $this->exportPdf($data);
            default:
                throw new \Exception('Unsupported export format');
        }
    }

    protected function exportCsv(array $data): string
    {
        $output = fopen('php://temp', 'w');

        // Header
        fputcsv($output, ['ID', 'Name', 'Email', 'Services', 'Total Spent', 'Joined']);

        // Data
        foreach ($data as $row) {
            fputcsv($output, [
                $row->id,
                $row->firstname . ' ' . $row->lastname,
                $row->email,
                $row->service_count,
                $row->total_spent ?? 0,
                $row->datecreated,
            ]);
        }

        rewind($output);
        return stream_get_contents($output);
    }
}
```

## Report Registration

```php
<?php
// /modules/reports/reports.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

return [
    'YourReport',
    'DetailedReport',
];
```

## Best Practices

1. **Date Filtering**: Always support date range filtering
2. **Pagination**: Implement pagination for large datasets
3. **Export Options**: Provide CSV, Excel, and PDF export
4. **Charts**: Include visualizations for better insights
5. **Permissions**: Implement appropriate access controls
6. **Caching**: Cache expensive queries appropriately
7. **Performance**: Optimize database queries
8. **Security**: Sanitize all output
9. **Summary Stats**: Include summary statistics
10. **Drill-down**: Support detailed views from summary data
