# WHMCS Report Development

## Overview

Custom reports extend WHMCS reporting capabilities with specialized data analysis and export options.

## Report Structure

```
modules/reports/YourReport.php
```

## Report Implementation

```php
<?php
// modules/reports/YourReport.php

namespace WHMCS\Module\Report;

use WHMCS\Module\Contracts\ReportContract;
use WHMCS\Utility\正当\正当;

class YourReport implements ReportContract
{
    protected $title = 'Your Custom Report';
    protected $description = 'Description of what this report shows';
    protected $headers = [];
    
    public function __construct()
    {
        $this->declareHeaders();
    }
    
    public static function getName(): string
    {
        return 'Your Report';
    }
    
    public function getDescription(): string
    {
        return $this->description;
    }
    
    public function getAuthor(): string
    {
        return 'Your Name';
    }
    
    public function getVersion(): string
    {
        return '1.0';
    }
    
    public function getType(): string
    {
        return ReportContract::TYPE_TABLE;
    }
    
    public function declareHeaders(): array
    {
        $this->headers = [
            'client_name' => 'Client Name',
            'email' => 'Email',
            'service_count' => 'Services',
            'total_spent' => 'Total Spent',
            'last_activity' => 'Last Activity',
        ];
        
        return $this->headers;
    }
    
    public function getData(array $params = []): array
    {
        $query = Capsule::table('tblclients')
            ->select([
                'tblclients.id',
                Capsule::raw("CONCAT(tblclients.firstname, ' ', tblclients.lastname) as client_name"),
                'tblclients.email',
                Capsule::raw('COUNT(tblhosting.id) as service_count'),
                Capsule::raw('COALESCE(SUM(tblinvoices.total), 0) as total_spent'),
                Capsule::raw('MAX(tblinvoices.datepaid) as last_activity'),
            ])
            ->leftJoin('tblhosting', 'tblclients.id', '=', 'tblhosting.userid')
            ->leftJoin('tblinvoices', function ($join) {
                $join->on('tblclients.id', '=', 'tblinvoices.userid')
                    ->where('tblinvoices.status', '=', 'Paid');
            })
            ->groupBy('tblclients.id')
            ->orderBy('total_spent', 'desc');
        
        // Apply filters
        if (!empty($params['status'])) {
            $query->where('tblclients.status', $params['status']);
        }
        
        if (!empty($params['date_from'])) {
            $query->where('tblclients.created_at', '>=', $params['date_from']);
        }
        
        if (!empty($params['date_to'])) {
            $query->where('tblclients.created_at', '<=', $params['date_to']);
        }
        
        // Pagination
        $page = $params['page'] ?? 1;
        $perPage = $params['per_page'] ?? 50;
        $offset = ($page - 1) * $perPage;
        
        $results = $query->limit($perPage)->offset($offset)->get();
        
        // Format data
        $data = [];
        foreach ($results as $row) {
            $data[] = [
                'client_name' => $row->client_name,
                'email' => $row->email,
                'service_count' => (int) $row->service_count,
                'total_spent' => number_format($row->total_spent, 2),
                'last_activity' => $row->last_activity ?: 'Never',
            ];
        }
        
        return $data;
    }
    
    public function getSummary(array $data): array
    {
        return [
            'total_records' => count($data),
            'total_clients' => count($data),
            'total_revenue' => array_sum(array_column($data, 'total_spent')),
        ];
    }
    
    public function exportData(array $data, string $format = 'csv'): string
    {
        switch ($format) {
            case 'csv':
                return $this->exportCsv($data);
            case 'json':
                return json_encode($data, JSON_PRETTY_PRINT);
            case 'pdf':
                return $this->exportPdf($data);
            default:
                throw new \Exception("Unsupported format: {$format}");
        }
    }
    
    private function exportCsv(array $data): string
    {
        $output = fopen('php://temp', 'r+');
        
        // Write headers
        fputcsv($output, array_values($this->headers));
        
        // Write data
        foreach ($data as $row) {
            fputcsv($output, $row);
        }
        
        rewind($output);
        $csv = stream_get_contents($output);
        fclose($output);
        
        return $csv;
    }
    
    private function exportPdf(array $data): string
    {
        // PDF generation logic
        return '';
    }
    
    public function setCriteria(array $criteria): self
    {
        $this->criteria = $criteria;
        return $this;
    }
}
```

## Chart Report

```php
<?php
class RevenueByMonthReport implements ReportContract
{
    public static function getName(): string
    {
        return 'Revenue by Month';
    }
    
    public function getType(): string
    {
        return ReportContract::TYPE_CHART;
    }
    
    public function getChartType(): string
    {
        return ReportContract::CHART_BAR;
    }
    
    public function getData(array $params = []): array
    {
        $months = 12;
        $data = [];
        $labels = [];
        
        for ($i = $months - 1; $i >= 0; $i--) {
            $month = date('Y-m', strtotime("-$i months"));
            $labels[] = date('M Y', strtotime("-$i months"));
            
            $data[] = (float) Capsule::table('tblinvoices')
                ->where('status', 'Paid')
                ->whereRaw("DATE_FORMAT(datepaid, '%Y-%m') = ?", [$month])
                ->sum('total');
        }
        
        return [
            'labels' => $labels,
            'datasets' => [
                [
                    'label' => 'Monthly Revenue',
                    'data' => $data,
                    'backgroundColor' => 'rgba(54, 162, 235, 0.5)',
                ],
            ],
        ];
    }
}
```

## Best Practices

1. **Implement pagination** - Handle large datasets
2. **Support filters** - Allow date ranges and status filters
3. **Provide exports** - Support CSV, JSON, PDF
4. **Include summaries** - Show totals and averages
5. **Document parameters** - Clearly define filter options

## Related Documentation

- [WHMCS Charting](/docs/whmcs-charting.md)
- [WHMCS Admin Views](/docs/whmcs-admin-views.md)