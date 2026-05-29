# WHMCS Report Automation Workflow

## Overview
This workflow implements automated report generation and scheduling for WHMCS.

## Prerequisites
- WHMCS with reporting access
- Database query capabilities
- Email configuration for report delivery

## Step-by-Step Process

### Step 1: Create Report Generator Class
```php
<?php
// /includes/reports/ReportGenerator.php

namespace WHMCS\Reports;

class ReportGenerator
{
    /**
     * Generate report and return data
     */
    public function generate(string $reportType, array $params = []): array
    {
        $method = "generate" . str_replace('_', '', ucwords($reportType, '_'));

        if (method_exists($this, $method)) {
            return $this->$method($params);
        }

        return ['error' => 'Unknown report type'];
    }

    /**
     * Get available reports
     */
    public function getAvailableReports(): array
    {
        return [
            'revenue' => 'Revenue Report',
            'sales' => 'Sales Report',
            'clients' => 'Client Report',
            'services' => 'Service Report',
            'invoices' => 'Invoice Report',
            'payments' => 'Payment Report',
            'tickets' => 'Support Tickets Report',
            'churn' => 'Churn Analysis',
            'mrr' => 'MRR Report',
            'arr' => 'ARR Report'
        ];
    }

    // Report generation methods
    private function generateRevenue(array $params): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');

        $data = Capsule::select("
            SELECT
                DATE(datepaid) as date,
                SUM(total) as revenue,
                COUNT(*) as transactions,
                paymentmethod
            FROM tblinvoices
            WHERE datepaid BETWEEN ? AND ?
            AND status = 'Paid'
            GROUP BY DATE(datepaid), paymentmethod
            ORDER BY date DESC
        ", [$startDate, $endDate]);

        return [
            'title' => 'Revenue Report',
            'period' => ['start' => $startDate, 'end' => $endDate],
            'total_revenue' => array_sum(array_column($data, 'revenue')),
            'total_transactions' => array_sum(array_column($data, 'transactions')),
            'data' => $data
        ];
    }

    private function generateClients(array $params): array
    {
        $data = Capsule::select("
            SELECT
                c.id, c.email, c.firstname, c.lastname, c.company,
                c.datecreated, c.datelastlogin, c.groupid,
                COUNT(h.id) as services,
                SUM(h.amount) as lifetime_value
            FROM tblclients c
            LEFT JOIN tblhosting h ON c.id = h.userid AND h.domainstatus = 'Active'
            GROUP BY c.id
            ORDER BY c.datecreated DESC
        ");

        return [
            'title' => 'Client Report',
            'total_clients' => count($data),
            'data' => $data
        ];
    }
}
```

### Step 2: Create Scheduled Reports
```php
<?php
// /includes/reports/ScheduledReports.php

class ScheduledReports
{
    private $reportGenerator;

    public function __construct()
    {
        $this->reportGenerator = new ReportGenerator();
    }

    /**
     * Schedule a report
     */
    public function schedule(string $reportType, string $frequency, array $recipients, array $params = []): int
    {
        $nextRun = $this->calculateNextRun($frequency);

        return Capsule::table('mod_scheduled_reports')->insertGetId([
            'report_type' => $reportType,
            'frequency' => $frequency,
            'recipients' => json_encode($recipients),
            'params' => json_encode($params),
            'next_run' => $nextRun,
            'status' => 'active',
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    /**
     * Process scheduled reports
     */
    public function processDueReports(): array
    {
        $dueReports = Capsule::table('mod_scheduled_reports')
            ->where('status', 'active')
            ->where('next_run', '<=', date('Y-m-d H:i:s'))
            ->get();

        $results = [];

        foreach ($dueReports as $report) {
            $result = $this->processReport($report);
            $results[] = $result;

            // Schedule next run
            $this->updateNextRun($report->id, $report->frequency);
        }

        return $results;
    }

    private function processReport($report): array
    {
        try {
            // Generate report
            $reportData = $this->reportGenerator->generate($report->report_type, json_decode($report->params, true));

            // Format report
            $formattedReport = $this->formatReport($reportData);

            // Send to recipients
            $recipients = json_decode($report->recipients, true);

            foreach ($recipients as $recipient) {
                sendEmail($recipient, 'Scheduled Report', [
                    'report_title' => $reportData['title'] ?? 'Report',
                    'report_content' => $formattedReport,
                    'period' => $reportData['period'] ?? null
                ]);
            }

            // Log success
            Capsule::table('mod_scheduled_reports_log')->insert([
                'report_id' => $report->id,
                'status' => 'success',
                'generated_at' => date('Y-m-d H:i:s')
            ]);

            return ['success' => true, 'report_id' => $report->id];
        } catch (Exception $e) {
            Capsule::table('mod_scheduled_reports_log')->insert([
                'report_id' => $report->id,
                'status' => 'failed',
                'error' => $e->getMessage(),
                'generated_at' => date('Y-m-d H:i:s')
            ]);

            return ['success' => false, 'report_id' => $report->id, 'error' => $e->getMessage()];
        }
    }

    private function formatReport(array $data): string
    {
        $html = "<h2>{$data['title']}</h2>";

        if (isset($data['period'])) {
            $html .= "<p>Period: {$data['period']['start']} to {$data['period']['end']}</p>";
        }

        $html .= "<table border='1' cellpadding='5' cellspacing='0'>";

        // Header
        if (!empty($data['data'])) {
            $headers = array_keys((array)$data['data'][0]);
            $html .= "<tr>";
            foreach ($headers as $header) {
                $html .= "<th>" . ucfirst(str_replace('_', ' ', $header)) . "</th>";
            }
            $html .= "</tr>";

            // Rows
            foreach ($data['data'] as $row) {
                $html .= "<tr>";
                foreach ((array)$row as $value) {
                    $html .= "<td>{$value}</td>";
                }
                $html .= "</tr>";
            }
        }

        $html .= "</table>";

        return $html;
    }

    private function calculateNextRun(string $frequency): string
    {
        switch ($frequency) {
            case 'daily':
                return date('Y-m-d H:i:s', strtotime('+1 day'));
            case 'weekly':
                return date('Y-m-d H:i:s', strtotime('+1 week'));
            case 'monthly':
                return date('Y-m-d H:i:s', strtotime('+1 month'));
            default:
                return date('Y-m-d H:i:s', strtotime('+1 day'));
        }
    }

    private function updateNextRun(int $reportId, string $frequency)
    {
        $nextRun = $this->calculateNextRun($frequency);

        Capsule::table('mod_scheduled_reports')
            ->where('id', $reportId)
            ->update(['next_run' => $nextRun]);
    }
}
```

### Step 3: Add Report Cron Hook
```php
<?php
// /includes/hooks/report_hooks.php

add_hook('DailyCronJob', 1, function($vars) {
    $scheduledReports = new ScheduledReports();
    $results = $scheduledReports->processDueReports();

    $successful = count(array_filter($results, fn($r) => $r['success']));
    $failed = count($results) - $successful;

    if ($failed > 0) {
        sendAdminEmail('Scheduled Report Errors', [
            'processed' => count($results),
            'successful' => $successful,
            'failed' => $failed
        ]);
    }

    return "Processed {$successful} reports successfully, {$failed} failed";
});
```

## Related Workflows
- [WHMCS Reporting & Analytics](../reporting/reporting-workflow.md)
- [WHMCS Revenue Report](./whmcs-revenue-report.md)