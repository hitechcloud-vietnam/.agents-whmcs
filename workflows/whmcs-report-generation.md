# WHMCS Report Generation Workflow
# Version: 1.0 | Created: 2026-05-28

## Purpose

Comprehensive guide to generating business reports in WHMCS including financial reports, service reports, client analytics, custom report development, and automated report scheduling.

## Prerequisites

- WHMCS installation with reporting module access
- Admin privileges for report generation
- Database query access for custom reports
- Email configuration for scheduled reports

## Workflow Steps

### Step 1: Understand WHMCS Report Structure

Access built-in WHMCS reports:

```php
// WHMCS built-in report locations
$reportPaths = [
    'admin'  => ROOTDIR . '/admin/reports/',
    'views'  => ROOTDIR . '/admin/reports/views/',
    'parser' => ROOTDIR . '/admin/reports/lib/',
];

// Key WHMCS reports available
$defaultReports = [
    'admin' => [
        'Open Tickets'      => 'opentickets.php',
        'Invoice List'     => 'invoicelist.php',
        'client_summary'   => 'client_summary.php',
        'product_summary'  => 'product_summary.php',
        'annual_income'    => 'annual_income_summary.php',
        '寝饉寞ﾗ '         => 'usageBilling.php',
    ],
];
```

### Step 2: Create Custom SQL Reports

Build custom reporting queries:

```php
// modules/addons/custom_reports/custom_reports.php

function custom_reports_config(): array
{
    return [
        'name'        => 'Custom Reports',
        'description' => 'Generate custom business reports',
        'version'     => '1.0',
    ];
}

function custom_reports_output(array $vars): void
{
    $reportType = $_REQUEST['type'] ?? 'dashboard';

    switch ($reportType) {
        case 'dashboard':
            include __DIR__ . '/views/dashboard.php';
            break;
        case 'revenue':
            include __DIR__ . '/reports/revenue_report.php';
            break;
        case 'churn':
            include __DIR__ . '/reports/churn_report.php';
            break;
        case 'usage':
            include __DIR__ . '/reports/usage_report.php';
            break;
    }
}

/**
 * Monthly Revenue Report
 */
function getMonthlyRevenueReport(string $startDate, string $endDate): array
{
    $query = "SELECT
                DATE_FORMAT(date, '%Y-%m') as month,
                COUNT(*) as transaction_count,
                SUM(amount) as total_revenue,
                SUM(amount - fees) as net_revenue,
                AVG(amount) as avg_transaction,
                MAX(amount) as max_transaction
              FROM tblaccounts
              WHERE date BETWEEN ? AND ?
              GROUP BY DATE_FORMAT(date, '%Y-%m')
              ORDER BY month DESC";

    $results = Capsule::select($query, [$startDate, $endDate]);

    return [
        'data'    => $results,
        'summary' => [
            'total_revenue'      => array_sum(array_column($results, 'total_revenue')),
            'total_transactions' => array_sum(array_column($results, 'transaction_count')),
            'avg_daily_revenue'  => array_sum(array_column($results, 'total_revenue')) / count($results),
        ],
    ];
}

/**
 * Service Health Report
 */
function getServiceHealthReport(): array
{
    $query = "SELECT
                p.name as product_name,
                COUNT(h.id) as total_services,
                SUM(CASE WHEN h.domainstatus = 'Active' THEN 1 ELSE 0 END) as active,
                SUM(CASE WHEN h.domainstatus = 'Suspended' THEN 1 ELSE 0 END) as suspended,
                SUM(CASE WHEN h.domainstatus = 'Terminated' THEN 1 ELSE 0 END) as terminated,
                SUM(CASE WHEN h.domainstatus = 'Canceled' THEN 1 ELSE 0 END) as canceled,
                AVG(TIMESTAMPDIFF(DAY, h.regdate, COALESCE(h.nextinvoicedate, NOW()))) as avg_age_days
              FROM tblhosting h
              JOIN tblproducts p ON h.packageid = p.id
              GROUP BY p.id
              ORDER BY total_services DESC";

    $results = Capsule::select($query);

    // Calculate churn rates
    foreach ($results as $product) {
        $total = $product->total_services;
        $churned = $product->terminated + $product->canceled;
        $product->churn_rate = ($total > 0) ? ($churned / $total) * 100 : 0;
    }

    return $results;
}

/**
 * Client Lifetime Value Report
 */
function getClientLifetimeValueReport(): array
{
    $query = "SELECT
                c.id as client_id,
                c.email,
                c.companyname,
                c.datecreated as client_since,
                COUNT(DISTINCT h.id) as service_count,
                COALESCE(SUM(a.amount), 0) as total_paid,
                COALESCE(SUM(a.amount), 0) / GROUPING(c.id) as arpu,
                MAX(a.date) as last_payment,
                TIMESTAMPDIFF(DAY, c.datecreated, NOW()) as client_age_days,
                tm.ticket_count,
                tm.open_tickets
              FROM tblclients c
              LEFT JOIN tblhosting h ON c.id = h.userid
              LEFT JOIN tblaccounts a ON c.id = a.userid
              LEFT JOIN (
                  SELECT userid,
                         COUNT(*) as ticket_count,
                         SUM(CASE WHEN status IN ('Open', 'Awaiting Reply') THEN 1 ELSE 0 END) as open_tickets
                  FROM tbltickets
                  GROUP BY userid
              ) tm ON c.id = tm.userid
              WHERE c.status = 'Active'
              GROUP BY c.id
              ORDER BY total_paid DESC
              LIMIT 100";

    return Capsule::select($query);
}
```

### Step 3: Build Report Views and Charts

Create visualization components:

```php
// modules/addons/custom_reports/views/dashboard.php

<div class="reports-dashboard">
    <div class="report-header">
        <h1>Business Reports</h1>
        <div class="date-range-selector">
            <form method="GET">
                <input type="hidden" name="module" value="custom_reports">
                <input type="hidden" name="type" value="<?= $reportType ?>">
                <input type="date" name="start_date" value="<?= $startDate ?>">
                <input type="date" name="end_date" value="<?= $endDate ?>">
                <button type="submit">Update</button>
            </form>
        </div>
    </div>

    <div class="summary-cards">
        <?php foreach ($summaryMetrics as $metric): ?>
        <div class="metric-card <?= $metric['trend'] ?>">
            <h3><?= $metric['label'] ?></h3>
            <p class="metric-value"><?= formatNumber($metric['value']) ?></p>
            <p class="metric-change">
                <?= $metric['change'] > 0 ? '+' : '' ?><?= $metric['change'] ?>%
                vs previous period
            </p>
        </div>
        <?php endforeach; ?>
    </div>

    <div class="charts-container">
        <div class="chart-panel">
            <h3>Revenue Trend</h3>
            <canvas id="revenueChart" width="800" height="300"></canvas>
        </div>

        <div class="chart-panel">
            <h3>Revenue by Product</h3>
            <canvas id="productPieChart" width="400" height="300"></canvas>
        </div>
    </div>

    <div class="data-table">
        <h3>Detailed Report</h3>
        <table class="report-table">
            <thead>
                <tr>
                    <?php foreach ($tableHeaders as $header): ?>
                    <th><?= $header ?></th>
                    <?php endforeach; ?>
                </tr>
            </thead>
            <tbody>
                <?php foreach ($tableData as $row): ?>
                <tr>
                    <?php foreach ($row as $cell): ?>
                    <td><?= $cell ?></td>
                    <?php endforeach; ?>
                </tr>
                <?php endforeach; ?>
            </tbody>
        </table>
    </div>
</div>

<script>
document.addEventListener('DOMContentLoaded', function() {
    const revenueCtx = document.getElementById('revenueChart').getContext('2d');
    new Chart(revenueCtx, {
        type: 'line',
        data: {
            labels: <?= json_encode($chartLabels) ?>,
            datasets: [{
                label: 'Revenue',
                data: <?= json_encode($chartData) ?>,
                borderColor: '#3498db',
                fill: true,
                backgroundColor: 'rgba(52, 152, 219, 0.1)',
            }]
        },
        options: {
            responsive: true,
            plugins: { legend: { display: false } },
            scales: {
                y: { beginAtZero: true, ticks: { callback: formatCurrency } }
            }
        }
    });
});
</script>
```

### Step 4: Implement Automated Report Scheduling

Schedule automated report generation:

```php
// includes/hooks/report_scheduler.php

add_hook('DailyCronJob', 1, function(array $vars) {
    $scheduler = new ReportScheduler();
    $scheduler->generateDailyReports();
});

add_hook('WeeklyCronJob', 1, function(array $vars) {
    $scheduler = new ReportScheduler();
    $scheduler->generateWeeklyReports();
});

add_hook('MonthlyCronRun', 1, function(array $vars) {
    $scheduler = new ReportScheduler();
    $scheduler->generateMonthlyReports();
});

class ReportScheduler
{
    private array $recipients = [
        'admin@yourcompany.com',
        'finance@yourcompany.com',
    ];

    public function generateDailyReports(): void
    {
        $date = date('Y-m-d');
        $yesterday = date('Y-m-d', strtotime('-1 day'));

        $reports = [
            'daily_sales' => $this->buildDailySalesReport($yesterday, $yesterday),
            'new_signups' => $this->buildNewSignupsReport($yesterday, $yesterday),
            'usage_summary' => $this->buildUsageSummary($yesterday),
        ];

        foreach ($this->recipients as $recipient) {
            $this->sendReportEmail($recipient, 'Daily Report - ' . $date, $reports);
        }
    }

    public function generateWeeklyReports(): void
    {
        $startDate = date('Y-m-d', strtotime('-7 days'));
        $endDate = date('Y-m-d');

        $reports = [
            'weekly_summary' => $this->buildWeeklySummary($startDate, $endDate),
            'churn_analysis' => $this->buildChurnAnalysis($startDate, $endDate),
            'support_metrics' => $this->buildSupportMetrics($startDate, $endDate),
        ];

        $emailBody = $this->buildWeeklyEmailBody($reports);

        foreach ($this->recipients as $recipient) {
            $this->sendHtmlEmail($recipient, 'Weekly Report', $emailBody);
        }
    }

    private function buildWeeklySummary(string $startDate, string $endDate): array
    {
        $revenue = Capsule::select(
            "SELECT SUM(amount) as total FROM tblaccounts WHERE date BETWEEN ? AND ?",
            [$startDate, $endDate]
        );

        $newClients = Capsule::table('tblclients')
            ->where('datecreated', '>=', $startDate)
            ->where('datecreated', '<=', $endDate)
            ->count();

        $newServices = Capsule::table('tblhosting')
            ->where('regdate', '>=', $startDate)
            ->where('regdate', '<=', $endDate)
            ->count();

        return [
            'period' => ['start' => $startDate, 'end' => $endDate],
            'revenue' => $revenue[0]->total ?? 0,
            'new_clients' => $newClients,
            'new_services' => $newServices,
        ];
    }

    private function sendReportEmail(string $recipient, string $subject, array $reports): void
    {
        $body = "Report Summary\n";
        $body .= "================\n\n";

        foreach ($reports as $name => $data) {
            $body .= "\n{$name}:\n";
            foreach ($data as $key => $value) {
                $body .= "  {$key}: {$value}\n";
            }
        }

        sendEmail([
            'to'      => $recipient,
            'subject' => $subject,
            'body'    => $body,
        ]);
    }

    private function sendHtmlEmail(string $recipient, string $subject, string $html): void
    {
        sendEmail([
            'to'       => $recipient,
            'subject'  => $subject,
            'body'     => $html,
            'fromname' => 'WHMCS Reports',
        ]);
    }
}
```

### Step 5: Export Reports to Multiple Formats

Implement support for various export formats:

```php
// modules/addons/custom_reports/export.php

class ReportExporter
{
    /**
     * Export to CSV format
     */
    public function exportToCsv(array $data, string $filename): void
    {
        header('Content-Type: text/csv');
        header('Content-Disposition: attachment; filename="' . $filename . '.csv"');

        $output = fopen('php://output', 'w');

        // Write headers
        if (!empty($data)) {
            fputcsv($output, array_keys((array) $data[0]));
        }

        // Write data rows
        foreach ($data as $row) {
            fputcsv($output, (array) $row);
        }

        fclose($output);
        exit;
    }

    /**
     * Export to Excel format
     */
    public function exportToExcel(array $data, string $filename): void
    {
        require_once __DIR__ . '/../lib/SimpleXLSX.php';

        $rows = [];

        // Headers
        if (!empty($data)) {
            $rows[] = array_keys((array) $data[0]);
        }

        // Data
        foreach ($data as $row) {
            $rows[] = array_values((array) $row);
        }

        $xlsx = new SimpleXLSX();
        $xlsx->addRows($rows);
        $xlsx->download($filename . '.xlsx');
    }

    /**
     * Export to PDF format
     */
    public function exportToPdf(array $data, string $title, array $options = []): void
    {
        require_once __DIR__ . '/../lib/TCPDF/tcpdf.php';

        $pdf = new TCPDF(PDF_PAGE_ORIENTATION, PDF_UNIT, PDF_PAGE_FORMAT, true, 'UTF-8');
        $pdf->SetCreator('WHMCS Reports');
        $pdf->SetTitle($title);

        $pdf->AddPage();
        $pdf->SetFont('helvetica', '', 10);

        // Title
        $pdf->Cell(0, 10, $title, 0, 1, 'C');
        $pdf->Ln(10);

        // Table header
        $headers = array_keys((array) $data[0]);
        $colWidths = $this->calculateColumnWidths($headers, $options['column_widths'] ?? []);

        $pdf->SetFont('helvetica', 'B', 9);
        foreach ($headers as $i => $header) {
            $pdf->Cell($colWidths[$i] ?? 50, 7, $header, 1, 0, 'C');
        }
        $pdf->Ln();

        // Table body
        $pdf->SetFont('helvetica', '', 9);
        foreach ($data as $row) {
            $values = array_values((array) $row);
            foreach ($values as $i => $value) {
                $pdf->Cell($colWidths[$i] ?? 50, 6, substr($value, 0, 50), 1, 0, 'L');
            }
            $pdf->Ln();
        }

        $pdf->Output($title . '.pdf', 'D');
    }
}

// Handle export requests
if (isset($_REQUEST['action']) && $_REQUEST['action'] === 'export') {
    check_token('WHMCS.admin.default');

    $reportData = getReportData($_REQUEST['report_type']);
    $exporter = new ReportExporter();

    switch ($_REQUEST['format']) {
        case 'csv':
            $exporter->exportToCsv($reportData, $_REQUEST['filename']);
            break;
        case 'excel':
            $exporter->exportToExcel($reportData, $_REQUEST['filename']);
            break;
        case 'pdf':
            $exporter->exportToPdf($reportData, $_REQUEST['filename']);
            break;
    }
}
```

---

## Best Practices

1. **Define report objectives** - Know what decisions each report supports
2. **Use consistent date ranges** - Compare similar time periods
3. **Include context** - Add benchmarks and comparisons
4. **Automate distribution** - Schedule reports for stakeholder delivery
5. **Secure sensitive data** - Control access to financial reports
6. **Document report definitions** - Maintain metadata for each report
7. **Test report accuracy** - Verify calculations against source data
8. **Optimize queries** - Cache expensive report queries
9. **Provide drill-down** - Allow users to explore details
10. **Archive old reports** - Maintain historical data for trends

---

## Verification Checklist

- [ ] Revenue report matches payment records
- [ ] Client counts accurate across reports
- [ ] Export formats produce valid files
- [ ] Scheduled reports sent on time
- [ ] Report access permissions enforced
- [ ] Daily/weekly/monthly reports generated correctly
- [ ] Charts render correct data
- [ ] PDF export formatting correct
- [ ] Report filters produce accurate results
- [ ] Historical data preserved for trend analysis
