# WHMCS Billing Reporting Workflow

## Purpose

Generate comprehensive billing reports, analyze financial data, track revenue metrics, and create exportable reports for business decision-making and compliance.

## Prerequisites

- WHMCS with billing data
- Admin access to reports section
- Export permissions configured
- Report scheduling capability (optional)

## Workflow Steps

### Step 1: Configure Report Settings

Set up reporting configuration:

```php
// Database configuration for reporting
INSERT INTO tblconfiguration (setting, value) VALUES 
('BillingReportEnabled', 'on'),
('DefaultReportPeriod', 'monthly'),
('ReportRetentionDays', '2555'),
('AutoEmailReports', 'on'),
('ReportAdminEmails', 'admin@example.com,accounting@example.com');

// Create reporting tables
Capsule::schema()->create('mod_billing_reports', function($t) {
    $t->increments('id');
    $t->string('report_name');
    $t->string('report_type');
    $t->date('start_date');
    $t->date('end_date');
    $t->json('parameters');
    $t->json('results');
    $t->integer('generated_by');
    $t->timestamp('created_at')->default(Capsule::raw('CURRENT_TIMESTAMP'));
});

Capsule::schema()->create('mod_revenue_tracking', function($t) {
    $t->increments('id');
    $t->date('period_date');
    $t->string('period_type'); // daily, monthly, quarterly
    $t->decimal('gross_revenue', 12, 2)->default(0);
    $t->decimal('refunds', 12, 2)->default(0);
    $t->decimal('discounts', 12, 2)->default(0);
    $t->decimal('net_revenue', 12, 2)->default(0);
    $t->decimal('tax_collected', 12, 2)->default(0);
    $t->integer('transaction_count')->default(0);
    $t->integer('new_customers')->default(0);
    $t->integer('churned_customers')->default(0);
    $t->timestamp('created_at')->default(Capsule::raw('CURRENT_TIMESTAMP'));
});
```

### Step 2: Create Reporting Service

Build reporting functionality:

```php
// File: /includes/classes/BillingReportService.php

namespace WHMCS\Reporting;

class BillingReportService {
    
    public function generateRevenueReport($startDate, $endDate, $groupBy = 'day') {
        $invoices = Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->whereBetween('datepaid', [$startDate, $endDate])
            ->get();
        
        $report = [
            'period' => ['start' => $startDate, 'end' => $endDate],
            'summary' => [
                'total_invoices' => 0,
                'gross_revenue' => 0,
                'refunds' => 0,
                'discounts' => 0,
                'net_revenue' => 0,
                'tax_collected' => 0,
                'average_invoice_value' => 0
            ],
            'by_payment_method' => [],
            'by_product_category' => [],
            'daily_breakdown' => []
        ];
        
        foreach ($invoices as $invoice) {
            $report['summary']['total_invoices']++;
            $report['summary']['gross_revenue'] += $invoice->total;
            $report['summary']['tax_collected'] += ($invoice->tax + $invoice->tax2);
            $report['summary']['net_revenue'] += $invoice->subtotal;
            
            // Group by payment method
            $method = $invoice->paymentmethod ?? 'unknown';
            if (!isset($report['by_payment_method'][$method])) {
                $report['by_payment_method'][$method] = ['count' => 0, 'amount' => 0];
            }
            $report['by_payment_method'][$method]['count']++;
            $report['by_payment_method'][$method]['amount'] += $invoice->total;
            
            // Group by day
            $day = date('Y-m-d', strtotime($invoice->datepaid));
            if (!isset($report['daily_breakdown'][$day])) {
                $report['daily_breakdown'][$day] = ['count' => 0, 'amount' => 0];
            }
            $report['daily_breakdown'][$day]['count']++;
            $report['daily_breakdown'][$day]['amount'] += $invoice->total;
        }
        
        $report['summary']['average_invoice_value'] = 
            $report['summary']['total_invoices'] > 0 
                ? round($report['summary']['gross_revenue'] / $report['summary']['total_invoices'], 2)
                : 0;
        
        return $report;
    }
    
    public function generateAgingReport() {
        $invoices = Capsule::table('tblinvoices')
            ->whereIn('status', ['Unpaid', 'Overdue'])
            ->where('total', '>', 0)
            ->get();
        
        $report = [
            'current' => ['count' => 0, 'amount' => 0],
            '1_30_days' => ['count' => 0, 'amount' => 0],
            '31_60_days' => ['count' => 0, 'amount' => 0],
            '61_90_days' => ['count' => 0, 'amount' => 0],
            'over_90_days' => ['count' => 0, 'amount' => 0],
            'total_outstanding' => 0
        ];
        
        $today = new \DateTime();
        
        foreach ($invoices as $invoice) {
            $dueDate = new \DateTime($invoice->duedate);
            $daysPastDue = $today->diff($dueDate)->days;
            
            if ($invoice->status === 'Unpaid' && $daysPastDue <= 0) {
                $report['current']['count']++;
                $report['current']['amount'] += $invoice->total;
            } elseif ($daysPastDue <= 30) {
                $report['1_30_days']['count']++;
                $report['1_30_days']['amount'] += $invoice->total;
            } elseif ($daysPastDue <= 60) {
                $report['31_60_days']['count']++;
                $report['31_60_days']['amount'] += $invoice->total;
            } elseif ($daysPastDue <= 90) {
                $report['61_90_days']['count']++;
                $report['61_90_days']['amount'] += $invoice->total;
            } else {
                $report['over_90_days']['count']++;
                $report['over_90_days']['amount'] += $invoice->total;
            }
            
            $report['total_outstanding'] += $invoice->total;
        }
        
        return $report;
    }
    
    public function generateMRRReport() {
        // Monthly Recurring Revenue
        $activeServices = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->get();
        
        $mrr = 0;
        $byProduct = [];
        
        foreach ($activeServices as $service) {
            $product = Capsule::table('tblproducts')
                ->where('id', $service->packageid)
                ->first();
            
            $billingCycle = $service->billingcycle;
            $amount = $product->recurringamount ?? 0;
            
            // Convert to monthly equivalent
            $monthlyAmount = $this->normalizeToMonthly($amount, $billingCycle);
            $mrr += $monthlyAmount;
            
            $category = $product->gid ?? 'uncategorized';
            if (!isset($byProduct[$category])) {
                $byProduct[$category] = ['count' => 0, 'mrr' => 0];
            }
            $byProduct[$category]['count']++;
            $byProduct[$category]['mrr'] += $monthlyAmount;
        }
        
        return [
            'mrr' => round($mrr, 2),
            'active_services' => count($activeServices),
            'by_category' => $byProduct,
            'calculated_at' => date('Y-m-d H:i:s')
        ];
    }
    
    private function normalizeToMonthly($amount, $cycle) {
        switch ($cycle) {
            case 'Monthly':
                return $amount;
            case 'Quarterly':
                return $amount / 3;
            case 'SemiAnnually':
                return $amount / 6;
            case 'Annually':
                return $amount / 12;
            case 'Biennially':
                return $amount / 24;
            default:
                return $amount;
        }
    }
    
    public function saveReport($name, $type, $startDate, $endDate, $results, $adminId) {
        return Capsule::table('mod_billing_reports')->insertGetId([
            'report_name' => $name,
            'report_type' => $type,
            'start_date' => $startDate,
            'end_date' => $endDate,
            'parameters' => json_encode(['group_by' => 'day']),
            'results' => json_encode($results),
            'generated_by' => $adminId,
            'created_at' => Capsule::raw('NOW()')
        ]);
    }
}
```

### Step 3: Create Report Generation Hooks

Schedule automated reports:

```php
// File: /includes/hooks/billing_reports.php

add_hook('DailyCronJob', 1, function($vars) {
    // Generate daily revenue snapshot
    $yesterday = date('Y-m-d', strtotime('-1 day'));
    $startDate = $yesterday . ' 00:00:00';
    $endDate = $yesterday . ' 23:59:59';
    
    $reportService = new \WHMCS\Reporting\BillingReportService();
    $report = $reportService->generateRevenueReport($startDate, $endDate);
    
    // Store daily snapshot
    Capsule::table('mod_revenue_tracking')->insert([
        'period_date' => $yesterday,
        'period_type' => 'daily',
        'gross_revenue' => $report['summary']['gross_revenue'],
        'refunds' => 0,
        'discounts' => 0,
        'net_revenue' => $report['summary']['net_revenue'],
        'tax_collected' => $report['summary']['tax_collected'],
        'transaction_count' => $report['summary']['total_invoices']
    ]);
    
    // Send daily summary to admin
    $summary = "Daily Revenue Report - {$yesterday}\n";
    $summary .= "Invoices: {$report['summary']['total_invoices']}\n";
    $summary .= "Gross: $" . number_format($report['summary']['gross_revenue'], 2) . "\n";
    $summary .= "Net: $" . number_format($report['summary']['net_revenue'], 2) . "\n";
    $summary .= "Tax: $" . number_format($report['summary']['tax_collected'], 2) . "\n";
    
    if ($report['summary']['gross_revenue'] > 0) {
        sendAdminNotification('email', 'Daily Revenue Report', $summary);
    }
});

add_hook('MonthlyCronJob', 1, function($vars) {
    // Generate monthly report on 1st of each month
    $lastMonth = date('Y-m-01', strtotime('-1 month'));
    $endDate = date('Y-m-t', strtotime('-1 month'));
    
    $reportService = new \WHMCS\Reporting\BillingReportService();
    $report = $reportService->generateRevenueReport($lastMonth, $endDate);
    
    // Save monthly report
    $reportService->saveReport(
        "Monthly Revenue Report - {$lastMonth}",
        'monthly_revenue',
        $lastMonth,
        $endDate,
        $report,
        $_SESSION['adminid'] ?? 0
    );
    
    // Generate and email monthly statement
    $emailReport = "Monthly Billing Report\n";
    $emailReport .= "Period: {$lastMonth} to {$endDate}\n\n";
    $emailReport .= "Summary:\n";
    foreach ($report['summary'] as $key => $value) {
        $label = ucwords(str_replace('_', ' ', $key));
        $emailReport .= "- {$label}: " . (is_numeric($value) ? '$' . number_format($value, 2) : $value) . "\n";
    }
    
    $emailReport .= "\nPayment Methods:\n";
    foreach ($report['by_payment_method'] as $method => $data) {
        $emailReport .= "- {$method}: {$data['count']} transactions, $" . number_format($data['amount'], 2) . "\n";
    }
    
    sendAdminNotification('email', "Monthly Billing Report - {$lastMonth}", $emailReport);
});
```

### Step 4: Create Report Export Functions

Implement CSV/Excel exports:

```php
// File: /includes/classes/ReportExporter.php

namespace WHMCS\Reporting;

class ReportExporter {
    
    public function exportToCSV($report, $filename) {
        $output = fopen('php://temp', 'r+');
        
        // Write headers
        if (isset($report['daily_breakdown'])) {
            fputcsv($output, ['Date', 'Invoice Count', 'Amount']);
            foreach ($report['daily_breakdown'] as $date => $data) {
                fputcsv($output, [$date, $data['count'], $data['amount']]);
            }
        } elseif (isset($report['by_payment_method'])) {
            fputcsv($output, ['Payment Method', 'Count', 'Amount']);
            foreach ($report['by_payment_method'] as $method => $data) {
                fputcsv($output, [$method, $data['count'], $data['amount']]);
            }
        }
        
        rewind($output);
        $csv = stream_get_contents($output);
        fclose($output);
        
        return $csv;
    }
    
    public function exportToJSON($report, $filename) {
        return json_encode($report, JSON_PRETTY_PRINT);
    }
    
    public function generatePDFReport($reportData) {
        // Implementation would use a PDF library like TCPDF or Dompdf
        // This is a placeholder for PDF generation
        
        $html = '<html><head><title>Billing Report</title></head><body>';
        $html .= '<h1>Billing Report</h1>';
        $html .= '<p>Period: ' . $reportData['period']['start'] . ' to ' . $reportData['period']['end'] . '</p>';
        $html .= '<h2>Summary</h2>';
        $html .= '<table>';
        foreach ($reportData['summary'] as $key => $value) {
            $html .= '<tr><td>' . ucfirst(str_replace('_', ' ', $key)) . '</td>';
            $html .= '<td>' . (is_numeric($value) ? '$' . number_format($value, 2) : $value) . '</td></tr>';
        }
        $html .= '</table></body></html>';
        
        return $html;
    }
}
```

### Step 5: Create Dashboard Widgets

Add report widgets to admin dashboard:

```php
// File: /includes/hooks/report_widgets.php

add_hook('AdminAreaHomepage', 1, function($vars) {
    $reportService = new \WHMCS\Reporting\BillingReportService();
    
    // Get current month data
    $startDate = date('Y-m-01');
    $endDate = date('Y-m-d');
    $revenue = $reportService->generateRevenueReport($startDate, $endDate . ' 23:59:59');
    
    // Get MRR
    $mrr = $reportService->generateMRRReport();
    
    // Get aging summary
    $aging = $reportService->generateAgingReport();
    
    return [
        'widgets' => [
            [
                'title' => 'Revenue This Month',
                'value' => '$' . number_format($revenue['summary']['gross_revenue'], 0),
                'subtitle' => $revenue['summary']['total_invoices'] . ' invoices',
                'icon' => 'fa-dollar-sign'
            ],
            [
                'title' => 'Monthly Recurring Revenue',
                'value' => '$' . number_format($mrr['mrr'], 0),
                'subtitle' => $mrr['active_services'] . ' active services',
                'icon' => 'fa-chart-line'
            ],
            [
                'title' => 'Outstanding Invoices',
                'value' => '$' . number_format($aging['total_outstanding'], 0),
                'subtitle' => 'Across all clients',
                'icon' => 'fa-exclamation-triangle'
            ]
        ]
    ];
});
```

## Verification Checklist

- [ ] Revenue reports generating correctly
- [ ] Aging report accurate with current data
- [ ] MRR calculation verified
- [ ] Daily reports scheduled and running
- [ ] Monthly reports generated on schedule
- [ ] CSV export working correctly
- [ ] JSON export working correctly
- [ ] Dashboard widgets displaying data
- [ ] Report storage configured
- [ ] Email notifications sent

## Related Skills and Documentation

- [WHMCS Invoice Automation](whmcs-invoice-automation-workflow.md)
- [WHMCS Payment Reconciliation](whmcs-payment-reconciliation-workflow.md)
- [WHMCS Billing Audit](whmcs-billing-audit-workflow.md)
- WHMCS Documentation: Reports
- WHMCS Documentation: Admin Dashboard

## Notes

- Schedule reports during off-peak hours
- Archive old reports to save storage
- Consider data retention policies
- Protect sensitive financial data
- Use secure delivery for sensitive reports
- Review reports regularly for anomalies
- Automate routine report generation