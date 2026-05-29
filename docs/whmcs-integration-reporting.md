# WHMCS Reporting Integration

Complete guide for generating and integrating reports.

## Overview

Create custom reports and integrate with analytics platforms.

## Report Generation

### Revenue Report

```php
<?php
/**
 * Generate revenue report
 */
function generateRevenueReport(string $startDate, string $endDate): array
{
    $invoices = Capsule::table('tblinvoices')
        ->whereBetween('date', [$startDate, $endDate])
        ->where('status', 'Paid')
        ->get();
    
    $report = [
        'period' => ['start' => $startDate, 'end' => $endDate],
        'generated_at' => date('Y-m-d H:i:s'),
        'summary' => [
            'total_revenue' => 0,
            'invoice_count' => 0,
            'average_invoice' => 0,
            'refunds' => 0,
        ],
        'by_product' => [],
        'by_payment_method' => [],
        'by_day' => [],
    ];
    
    foreach ($invoices as $invoice) {
        $report['summary']['total_revenue'] += $invoice->total;
        $report['summary']['invoice_count']++;
        
        // Get invoice items
        $items = Capsule::table('tblinvoiceitems')
            ->where('invoiceid', $invoice->id)
            ->get();
        
        foreach ($items as $item) {
            if ($item->type === 'Hosting') {
                $product = Capsule::table('tblhosting')
                    ->where('id', $item->relid)
                    ->first();
                
                if ($product) {
                    $productName = $product->product->name ?? 'Unknown';
                    if (!isset($report['by_product'][$productName])) {
                        $report['by_product'][$productName] = 0;
                    }
                    $report['by_product'][$productName] += $item->amount;
                }
            }
        }
        
        // By payment method
        $paymentMethod = $invoice->paymentmethod ?? 'unknown';
        if (!isset($report['by_payment_method'][$paymentMethod])) {
            $report['by_payment_method'][$paymentMethod] = 0;
        }
        $report['by_payment_method'][$paymentMethod] += $invoice->total;
        
        // By day
        $day = date('Y-m-d', strtotime($invoice->date));
        if (!isset($report['by_day'][$day])) {
            $report['by_day'][$day] = 0;
        }
        $report['by_day'][$day] += $invoice->total;
    }
    
    // Calculate averages
    if ($report['summary']['invoice_count'] > 0) {
        $report['summary']['average_invoice'] = 
            $report['summary']['total_revenue'] / $report['summary']['invoice_count'];
    }
    
    // Get refunds
    $refunds = Capsule::table('tblaccounts')
        ->whereBetween('date', [$startDate, $endDate])
        ->where('type', 'Out')
        ->sum('amount');
    
    $report['summary']['refunds'] = abs($refunds);
    $report['summary']['net_revenue'] = $report['summary']['total_revenue'] - $report['summary']['refunds'];
    
    return $report;
}
```

### Service Report

```php
<?php
/**
 * Generate service statistics report
 */
function generateServiceReport(): array
{
    $report = [
        'generated_at' => date('Y-m-d H:i:s'),
        'totals' => [
            'total_services' => 0,
            'active' => 0,
            'suspended' => 0,
            'terminated' => 0,
            'pending' => 0,
        ],
        'by_product' => [],
        'expiring_soon' => [],
        'recent_signups' => [],
    ];
    
    // Get all services
    $services = Capsule::table('tblhosting')->get();
    $report['totals']['total_services'] = count($services);
    
    foreach ($services as $service) {
        // Count by status
        $status = strtolower($service->domainstatus);
        if (isset($report['totals'][$status])) {
            $report['totals'][$status]++;
        }
        
        // By product
        $productName = $service->product->name ?? 'Unknown';
        if (!isset($report['by_product'][$productName])) {
            $report['by_product'][$productName] = [
                'count' => 0,
                'active' => 0,
                'monthly_revenue' => 0,
            ];
        }
        $report['by_product'][$productName]['count']++;
        
        if ($status === 'active') {
            $report['by_product'][$productName]['active']++;
            $report['by_product'][$productName]['monthly_revenue'] += 
                $service->billingcycle === 'Monthly' ? $service->amount : 0;
        }
        
        // Expiring within 30 days
        $daysUntilExpiry = (strtotime($service->nextduedate) - time()) / 86400;
        if ($daysUntilExpiry > 0 && $daysUntilExpiry <= 30) {
            $report['expiring_soon'][] = [
                'service_id' => $service->id,
                'domain' => $service->domain,
                'expires' => $service->nextduedate,
                'days_left' => round($daysUntilExpiry),
            ];
        }
        
        // Recent signups (last 7 days)
        $daysSinceSignup = (time() - strtotime($service->regdate)) / 86400;
        if ($daysSinceSignup <= 7) {
            $report['recent_signups'][] = [
                'service_id' => $service->id,
                'domain' => $service->domain,
                'signup_date' => $service->regdate,
            ];
        }
    }
    
    return $report;
}
```

## Analytics Integration

### Google Analytics

```php
<?php
/**
 * Send events to Google Analytics
 */
class GoogleAnalyticsClient
{
    private string $measurementId;
    private string $apiSecret;
    
    public function __construct(string $measurementId, string $apiSecret)
    {
        $this->measurementId = $measurementId;
        $this->apiSecret = $apiSecret;
    }
    
    /**
     * Send event
     */
    public function sendEvent(string $clientId, string $category, string $action, array $properties = []): bool
    {
        $data = [
            'client_id' => $clientId,
            'events' => [[
                'name' => $action,
                'params' => array_merge([
                    'category' => $category,
                ], $properties),
            ]],
        ];
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => "https://www.google-analytics.com/mp/collect?measurement_id={$this->measurementId}&api_secret={$this->apiSecret}",
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return $httpCode === 204;
    }
    
    /**
     * Track purchase
     */
    public function trackPurchase(string $clientId, array $invoice): bool
    {
        $data = [
            'client_id' => $clientId,
            'events' => [[
                'name' => 'purchase',
                'params' => [
                    'transaction_id' => $invoice['id'],
                    'value' => $invoice['total'],
                    'currency' => 'USD',
                    'tax' => $invoice['tax'],
                ],
            ]],
        ];
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => "https://www.google-analytics.com/mp/collect?measurement_id={$this->measurementId}&api_secret={$this->apiSecret}",
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        curl_exec($ch);
        curl_close($ch);
        
        return true;
    }
}
```

### Analytics Hooks

```php
<?php
/**
 * Track client registrations
 */
add_hook('ClientAdd', 1, function($vars) {
    $ga = new GoogleAnalyticsClient(GA_MEASUREMENT_ID, GA_API_SECRET);
    
    $ga->sendEvent($vars['userid'], 'registration', 'new_client', [
        'registration_method' => 'standard',
    ]);
});

/**
 * Track purchases
 */
add_hook('InvoicePaid', 1, function($vars) {
    $ga = new GoogleAnalyticsClient(GA_MEASUREMENT_ID, GA_API_SECRET);
    
    $invoice = Capsule::table('tblinvoices')
        ->where('id', $vars['invoice_id'])
        ->first();
    
    $ga->trackPurchase($vars['user_id'], (array)$invoice);
});

/**
 * Track service orders
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    $ga = new GoogleAnalyticsClient(GA_MEASUREMENT_ID, GA_API_SECRET);
    
    $service = Capsule::table('tblhosting')
        ->where('id', $vars['serviceid'])
        ->first();
    
    $ga->sendEvent($vars['params']['userid'], 'service', 'new_subscription', [
        'product_id' => $service->packageid,
        'product_name' => $service->product->name ?? '',
        'billing_cycle' => $service->billingcycle,
    ]);
});
```

## Report Export

```php
<?php
/**
 * Export report to CSV
 */
function exportReportToCSV(array $report, string $filename): string
{
    $path = sys_get_temp_dir() . '/' . $filename . '.csv';
    $fp = fopen($path, 'w');
    
    // Write summary
    fputcsv($fp, ['Summary Report']);
    foreach ($report['summary'] ?? [] as $key => $value) {
        fputcsv($fp, [ucwords(str_replace('_', ' ', $key)), $value]);
    }
    
    fputcsv($fp, []); // Empty row
    
    // Write by product
    fputcsv($fp, ['By Product']);
    fputcsv($fp, ['Product', 'Count', 'Active', 'Monthly Revenue']);
    foreach ($report['by_product'] ?? [] as $product => $data) {
        fputcsv($fp, [$product, $data['count'], $data['active'], $data['monthly_revenue']]);
    }
    
    fputcsv($fp, []);
    
    // Write daily breakdown
    fputcsv($fp, ['Daily Revenue']);
    fputcsv($fp, ['Date', 'Revenue']);
    foreach ($report['by_day'] ?? [] as $date => $revenue) {
        fputcsv($fp, [$date, $revenue]);
    }
    
    fclose($fp);
    
    return $path;
}

/**
 * Export report to PDF
 */
function exportReportToPDF(array $report, string $filename): string
{
    // Using a PDF library
    require_once __DIR__ . '/lib/tcpdf/tcpdf.php';
    
    $pdf = new TCPDF();
    $pdf->AddPage();
    
    // Title
    $pdf->SetFont('helvetica', 'B', 16);
    $pdf->Cell(0, 10, 'Revenue Report', 0, 1, 'C');
    
    // Period
    $pdf->SetFont('helvetica', '', 12);
    $pdf->Cell(0, 10, "Period: {$report['period']['start']} to {$report['period']['end']}", 0, 1);
    
    // Summary
    $pdf->SetFont('helvetica', 'B', 14);
    $pdf->Cell(0, 10, 'Summary', 0, 1);
    
    $pdf->SetFont('helvetica', '', 12);
    foreach ($report['summary'] as $key => $value) {
        $label = ucwords(str_replace('_', ' ', $key));
        $pdf->Cell(60, 8, $label . ':', 0, 0);
        $pdf->Cell(0, 8, is_numeric($value) ? number_format($value, 2) : $value, 0, 1);
    }
    
    // By Product
    $pdf->AddPage();
    $pdf->SetFont('helvetica', 'B', 14);
    $pdf->Cell(0, 10, 'Revenue by Product', 0, 1);
    
    $pdf->SetFont('helvetica', '', 12);
    foreach ($report['by_product'] as $product => $revenue) {
        $pdf->Cell(100, 8, $product, 0, 0);
        $pdf->Cell(0, 8, '$' . number_format($revenue, 2), 0, 1);
    }
    
    $path = sys_get_temp_dir() . '/' . $filename . '.pdf';
    $pdf->Output($path, 'F');
    
    return $path;
}
```

## Scheduled Reports

```php
<?php
/**
 * Generate and email scheduled report
 */
function sendScheduledReport(): void
{
    $reportConfig = Capsule::table('mod_scheduled_reports')
        ->where('name', 'weekly_revenue')
        ->where('enabled', 1)
        ->first();
    
    if (!$reportConfig) return;
    
    $startDate = date('Y-m-d', strtotime('-7 days'));
    $endDate = date('Y-m-d');
    
    $report = generateRevenueReport($startDate, $endDate);
    
    // Generate attachment
    $csvPath = exportReportToCSV($report, 'weekly_revenue_' . date('Y-m-d'));
    
    // Send email
    $email = new SendGridEmailClient(SENDGRID_API_KEY);
    $email->send([
        'to' => [$reportConfig->recipients],
        'from_email' => 'reports@example.com',
        'from_name' => 'WHMCS Reports',
        'subject' => 'Weekly Revenue Report - ' . date('Y-m-d'),
        'body' => 'Please find attached the weekly revenue report.',
    ], [$csvPath]);
    
    // Update last run
    Capsule::table('mod_scheduled_reports')
        ->where('id', $reportConfig->id)
        ->update(['last_run' => date('Y-m-d H:i:s')]);
}
```

## Best Practices

1. **Efficient queries** - Optimize database queries
2. **Cache reports** - Store generated reports
3. **Schedule wisely** - Run heavy reports off-peak
4. **Multiple formats** - Support CSV, PDF, Excel
5. **Track access** - Log report views
6. **Compress data** - Reduce storage needs

## Related Documentation

- [whmcs-integration-api.md](whmcs-integration-api.md)
- [whmcs-integration-analytics.md](whmcs-integration-analytics.md)
