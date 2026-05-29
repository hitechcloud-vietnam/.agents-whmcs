# WHMCS Report Module Builder

## Concept

Report modules add custom reporting capabilities to WHMCS. They can generate detailed reports, export data, and display statistics in the admin reports section.

## File Structure

```
/modules/reports/
├── yourreport.php    # Report module
└── templates/
    └── yourreport.tpl
```

## Core Report Structure

```php
<?php
/**
 * Report Module: Your Report
 * Version: 1.0.0
 * Description: Custom report for...
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Module\Report\ReportInterface;

if (interface_exists('WHMCS\Module\Report\ReportInterface')) {
    class YourReport implements ReportInterface
    {
        protected $exported = false;
        
        public function getName()
        {
            return 'Your Custom Report';
        }
        
        public function getDescription()
        {
            return 'Generate custom reports for your business';
        }
        
        public function getOptions()
        {
            return [
                'date_range' => [
                    'FriendlyName' => 'Date Range',
                    'Type' => 'dropdown',
                    'Options' => [
                        'today' => 'Today',
                        'week' => 'This Week',
                        'month' => 'This Month',
                        'quarter' => 'This Quarter',
                        'custom' => 'Custom Range',
                    ],
                    'Default' => 'month',
                ],
                'group_by' => [
                    'FriendlyName' => 'Group By',
                    'Type' => 'dropdown',
                    'Options' => [
                        'day' => 'Day',
                        'week' => 'Week',
                        'month' => 'Month',
                        'client' => 'Client',
                    ],
                ],
                'include_cancelled' => [
                    'FriendlyName' => 'Include Cancelled',
                    'Type' => 'yesno',
                ],
            ];
        }
        
        public function getResults(array $options = [])
        {
            $dateRange = $this->getDateRange($options['date_range']);
            $groupBy = $options['group_by'] ?? 'day';
            
            $results = $this->fetchData($dateRange, $groupBy, $options);
            
            return [
                'success' => true,
                'data' => $results,
                'totals' => $this->calculateTotals($results),
                'metadata' => [
                    'date_range' => $dateRange,
                    'group_by' => $groupBy,
                    'generated_at' => date('Y-m-d H:i:s'),
                ],
            ];
        }
        
        protected function getDateRange(string $range)
        {
            $end = date('Y-m-d 23:59:59');
            
            switch ($range) {
                case 'today':
                    $start = date('Y-m-d 00:00:00');
                    break;
                case 'week':
                    $start = date('Y-m-d 00:00:00', strtotime('-1 week'));
                    break;
                case 'quarter':
                    $start = date('Y-m-d 00:00:00', strtotime('-3 months'));
                    break;
                case 'custom':
                    $start = date('Y-m-d 00:00:00', strtotime($_GET['start_date'] ?? '-1 month'));
                    $end = date('Y-m-d 23:59:59', strtotime($_GET['end_date'] ?? 'now'));
                    break;
                case 'month':
                default:
                    $start = date('Y-m-d 00:00:00', strtotime('-1 month'));
                    break;
            }
            
            return ['start' => $start, 'end' => $end];
        }
        
        protected function fetchData(array $dateRange, string $groupBy, array $options)
        {
            $query = Capsule::table('tblhosting')
                ->selectRaw("
                    DATE(created) as date,
                    COUNT(*) as total_count,
                    SUM(amount) as total_amount
                ")
                ->whereBetween('created', [$dateRange['start'], $dateRange['end']])
                ->groupBy('date')
                ->orderBy('date');
            
            if (empty($options['include_cancelled'])) {
                $query->where('domainstatus', 'Active');
            }
            
            return $query->get()->toArray();
        }
        
        protected function calculateTotals(array $results)
        {
            $totals = [
                'count' => 0,
                'amount' => 0,
            ];
            
            foreach ($results as $row) {
                $totals['count'] += $row->total_count;
                $totals['amount'] += $row->total_amount;
            }
            
            return $totals;
        }
        
        public function exportData(array $options = [])
        {
            $this->exported = true;
            $results = $this->getResults($options);
            
            $filename = 'report_' . date('Y-m-d') . '.csv';
            
            header('Content-Type: text/csv');
            header('Content-Disposition: attachment; filename="' . $filename . '"');
            
            $output = fopen('php://output', 'w');
            
            // Header row
            fputcsv($output, ['Date', 'Count', 'Amount']);
            
            // Data rows
            foreach ($results['data'] as $row) {
                fputcsv($output, [
                    $row->date,
                    $row->total_count,
                    number_format($row->total_amount, 2),
                ]);
            }
            
            // Totals row
            fputcsv($output, [
                'TOTAL',
                $results['totals']['count'],
                number_format($results['totals']['amount'], 2),
            ]);
            
            fclose($output);
            exit;
        }
        
        public function listCategories()
        {
            return [
                'sales' => 'Sales Reports',
                'clients' => 'Client Reports',
                'services' => 'Service Reports',
            ];
        }
    }
} else {
    // Legacy report module format
    function yourreport_report()
    {
        return [
            'title' => 'Your Custom Report',
            'description' => 'Generate custom reports',
            'category' => 'sales',
        ];
    }
    
    function yourreport_results()
    {
        // Fetch and return report data
        $data = [];
        
        return [
            'data' => $data,
            'download' => true,
        ];
    }
}
```

## Template File

```smarty
<div class="report-container">
    <div class="report-header">
        <h2>{$report.title}</h2>
        <p class="text-muted">Generated: {$report.generated_at}</p>
    </div>
    
    <div class="report-filters">
        <form method="get" class="form-inline">
            <input type="hidden" name="module" value="reports">
            <input type="hidden" name="action" value="generate">
            <input type="hidden" name="report" value="yourreport">
            
            <div class="form-group">
                <label>Date Range</label>
                <select name="date_range" class="form-control">
                    <option value="today" {if $options.date_range eq 'today'}selected{/if}>Today</option>
                    <option value="week" {if $options.date_range eq 'week'}selected{/if}>This Week</option>
                    <option value="month" {if $options.date_range eq 'month'}selected{/if}>This Month</option>
                    <option value="quarter" {if $options.date_range eq 'quarter'}selected{/if}>This Quarter</option>
                </select>
            </div>
            
            <button type="submit" class="btn btn-primary">Generate Report</button>
            <a href="?module=reports&action=export&report=yourreport" class="btn btn-default">
                <i class="fa fa-download"></i> Export CSV
            </a>
        </form>
    </div>
    
    <div class="report-summary">
        <div class="row">
            <div class="col-md-4">
                <div class="summary-box">
                    <div class="summary-value">{$report.totals.count}</div>
                    <div class="summary-label">Total Items</div>
                </div>
            </div>
            <div class="col-md-4">
                <div class="summary-box">
                    <div class="summary-value">{$report.totals.amount|string_format:"%.2f"}</div>
                    <div class="summary-label">Total Amount</div>
                </div>
            </div>
        </div>
    </div>
    
    <div class="report-table">
        <table class="table table-striped">
            <thead>
                <tr>
                    <th>Date</th>
                    <th>Count</th>
                    <th>Amount</th>
                </tr>
            </thead>
            <tbody>
                {foreach $report.data as $row}
                <tr>
                    <td>{$row.date}</td>
                    <td>{$row.count}</td>
                    <td>{$row.amount|string_format:"%.2f"}</td>
                </tr>
                {/foreach}
            </tbody>
        </table>
    </div>
</div>
```

## Step-by-Step Implementation

1. Create report file in `/modules/reports/`
2. Implement ReportInterface or legacy format
3. Define report options
4. Implement getResults with data fetching
5. Implement export functionality
6. Create report template
7. Test in reports section

## Implementation Checklist

- [ ] Create report file in modules/reports
- [ ] Implement ReportInterface or legacy format
- [ ] Define getName and getDescription
- [ ] Define report options
- [ ] Implement getResults method
- [ ] Add date range filtering
- [ ] Implement calculateTotals
- [ ] Implement exportData for CSV
- [ ] Create template file
- [ ] Test in reports section