# WHMCS Billing Report Module DevKit

## Purpose
Comprehensive billing and finance reporting system with customizable reports, charts, exports, and scheduled report delivery.

## Module Type
Reporting/Analytics Addon Module

## Use Cases
- Revenue reports and trends
- Invoice aging analysis
- Payment method breakdown
- Refund and credit analysis
- Client billing summaries
- Tax reporting
- Commission calculations

## Database Schema

```sql
-- Report configurations (saved reports)
CREATE TABLE `mod_billing_reports` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `name` VARCHAR(100) NOT NULL,
  `description` TEXT NULL,
  `report_type` VARCHAR(50) NOT NULL,
  `config` TEXT NOT NULL COMMENT 'JSON configuration',
  `columns` TEXT NULL COMMENT 'JSON column selection',
  `filters` TEXT NULL COMMENT 'JSON filter settings',
  `sort_order` VARCHAR(50) NULL,
  `group_by` VARCHAR(50) NULL,
  `chart_type` VARCHAR(20) NULL,
  `is_shared` TINYINT(1) NOT NULL DEFAULT 0,
  `is_favorite` TINYINT(1) NOT NULL DEFAULT 0,
  `created_by` INT UNSIGNED NOT NULL,
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (`created_by`) REFERENCES `tbladmins`(`id`) ON DELETE CASCADE,
  INDEX `idx_type` (`report_type`),
  INDEX `idx_creator` (`created_by`)
) ENGINE=InnoDB;

-- Scheduled reports
CREATE TABLE `mod_report_schedules` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `report_id` INT UNSIGNED NOT NULL,
  `schedule_type` ENUM('daily', 'weekly', 'monthly', 'quarterly') NOT NULL,
  `send_day` TINYINT UNSIGNED NULL COMMENT 'Day of week/month',
  `send_time` TIME NOT NULL DEFAULT '09:00:00',
  `recipients` TEXT NOT NULL COMMENT 'JSON array of emails',
  `format` ENUM('pdf', 'csv', 'excel', 'html') NOT NULL DEFAULT 'pdf',
  `is_active` TINYINT(1) NOT NULL DEFAULT 1,
  `last_run` DATETIME NULL,
  `next_run` DATETIME NULL,
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (`report_id`) REFERENCES `mod_billing_reports`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB;

-- Report execution logs
CREATE TABLE `mod_report_logs` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `report_id` INT UNSIGNED NOT NULL,
  `schedule_id` INT UNSIGNED NULL,
  `status` ENUM('running', 'completed', 'failed') NOT NULL,
  `execution_time` INT UNSIGNED NULL COMMENT 'Seconds',
  `row_count` INT UNSIGNED NULL,
  `file_size` INT UNSIGNED NULL COMMENT 'Bytes',
  `error_message` TEXT NULL,
  `executed_by` INT UNSIGNED NULL,
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (`report_id`) REFERENCES `mod_billing_reports`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB;

-- Cached report data for faster loading
CREATE TABLE `mod_report_cache` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `report_id` INT UNSIGNED NOT NULL,
  `cache_key` VARCHAR(64) NOT NULL,
  `data` LONGTEXT NOT NULL,
  `expires_at` DATETIME NOT NULL,
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (`report_id`) REFERENCES `mod_billing_reports`(`id`) ON DELETE CASCADE,
  UNIQUE KEY `uk_report_cache` (`report_id`, `cache_key`)
) ENGINE=InnoDB;
```

## Code Template

### Main Module File (billing_report.php)

```php
<?php
/**
 * WHMCS Billing Report Module
 *
 * @package WHMCS
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Module configuration
 */
function billing_report_config()
{
    return [
        'name' => 'Billing Reports',
        'description' => 'Comprehensive billing and finance reporting',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'cacheEnabled' => [
                'Type' => 'yesno',
                'Default' => 'on',
                'Description' => 'Enable report caching',
            ],
            'cacheMinutes' => [
                'Type' => 'text',
                'Default' => '30',
                'Description' => 'Cache expiration (minutes)',
            ],
            'maxExportRows' => [
                'Type' => 'text',
                'Default' => '50000',
                'Description' => 'Maximum rows for export',
            ],
            'enableScheduling' => [
                'Type' => 'yesno',
                'Default' => 'on',
                'Description' => 'Enable scheduled reports',
            ],
        ],
    ];
}

/**
 * Activate module
 */
function billing_report_activate()
{
    $queries = [
        "CREATE TABLE IF NOT EXISTS `mod_billing_reports` (
            `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
            `name` VARCHAR(100) NOT NULL,
            `description` TEXT NULL,
            `report_type` VARCHAR(50) NOT NULL,
            `config` TEXT NOT NULL,
            `columns` TEXT NULL,
            `filters` TEXT NULL,
            `sort_order` VARCHAR(50) NULL,
            `group_by` VARCHAR(50) NULL,
            `chart_type` VARCHAR(20) NULL,
            `is_shared` TINYINT(1) NOT NULL DEFAULT 0,
            `is_favorite` TINYINT(1) NOT NULL DEFAULT 0,
            `created_by` INT UNSIGNED NOT NULL,
            `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
            `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
            INDEX `idx_type` (`report_type`)
        ) ENGINE=InnoDB",

        "CREATE TABLE IF NOT EXISTS `mod_report_schedules` (
            `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
            `report_id` INT UNSIGNED NOT NULL,
            `schedule_type` ENUM('daily', 'weekly', 'monthly', 'quarterly') NOT NULL,
            `send_day` TINYINT UNSIGNED NULL,
            `send_time` TIME NOT NULL DEFAULT '09:00:00',
            `recipients` TEXT NOT NULL,
            `format` ENUM('pdf', 'csv', 'excel', 'html') NOT NULL DEFAULT 'pdf',
            `is_active` TINYINT(1) NOT NULL DEFAULT 1,
            `last_run` DATETIME NULL,
            `next_run` DATETIME NULL,
            FOREIGN KEY (`report_id`) REFERENCES `mod_billing_reports`(`id`) ON DELETE CASCADE
        ) ENGINE=InnoDB",

        "CREATE TABLE IF NOT EXISTS `mod_report_logs` (
            `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
            `report_id` INT UNSIGNED NOT NULL,
            `schedule_id` INT UNSIGNED NULL,
            `status` ENUM('running', 'completed', 'failed') NOT NULL,
            `execution_time` INT UNSIGNED NULL,
            `row_count` INT UNSIGNED NULL,
            `error_message` TEXT NULL,
            `executed_by` INT UNSIGNED NULL,
            `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (`report_id`) REFERENCES `mod_billing_reports`(`id`) ON DELETE CASCADE
        ) ENGINE=InnoDB",

        "CREATE TABLE IF NOT EXISTS `mod_report_cache` (
            `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
            `report_id` INT UNSIGNED NOT NULL,
            `cache_key` VARCHAR(64) NOT NULL,
            `data` LONGTEXT NOT NULL,
            `expires_at` DATETIME NOT NULL,
            `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (`report_id`) REFERENCES `mod_billing_reports`(`id`) ON DELETE CASCADE,
            UNIQUE KEY `uk_report_cache` (`report_id`, `cache_key`)
        ) ENGINE=InnoDB"
    ];

    try {
        foreach ($queries as $query) {
            full_query($query);
        }

        // Insert default reports
        $defaultReports = [
            [
                'name' => 'Revenue Summary',
                'description' => 'Monthly revenue breakdown by product',
                'report_type' => 'revenue',
                'config' => json_encode(['period' => 'monthly']),
            ],
            [
                'name' => 'Invoice Aging',
                'description' => 'Outstanding invoices by age',
                'report_type' => 'aging',
                'config' => json_encode(['aging_buckets' => '30,60,90']),
            ],
            [
                'name' => 'Payment Methods',
                'description' => 'Payments breakdown by gateway',
                'report_type' => 'payments',
                'config' => json_encode(['group_by' => 'gateway']),
            ],
        ];

        foreach ($defaultReports as $report) {
            full_query("INSERT IGNORE INTO `mod_billing_reports` (`name`, `description`, `report_type`, `config`, `created_by`) VALUES (
                '" . db_escape_string($report['name']) . "',
                '" . db_escape_string($report['description']) . "',
                '" . db_escape_string($report['report_type']) . "',
                '" . db_escape_string($report['config']) . "',
                1
            )");
        }

        return ['status' => 'success', 'description' => 'Billing Report module activated'];
    } catch (Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

/**
 * Deactivate module
 */
function billing_report_deactivate()
{
    try {
        full_query("DROP TABLE IF EXISTS `mod_report_cache`");
        full_query("DROP TABLE IF EXISTS `mod_report_logs`");
        full_query("DROP TABLE IF EXISTS `mod_report_schedules`");
        full_query("DROP TABLE IF EXISTS `mod_billing_reports`");
        return ['status' => 'success'];
    } catch (Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}
```

### Billing Report Service Class

```php
<?php
/**
 * Billing Report Service Class
 */

namespace WHMCS\Module\BillingReport;

use WHMCS\Database\Capsule;

class BillingReportService
{
    protected $cacheEnabled = true;
    protected $cacheMinutes = 30;

    public function __construct()
    {
        $config = Capsule::table('mod_billing_report_config')->first();
        if ($config) {
            $this->cacheEnabled = $config->cache_enabled;
            $this->cacheMinutes = $config->cache_minutes ?? 30;
        }
    }

    /**
     * Generate revenue report
     */
    public function generateRevenueReport($filters = [])
    {
        $cacheKey = $this->generateCacheKey('revenue', $filters);
        if ($this->cacheEnabled) {
            $cached = $this->getFromCache($cacheKey);
            if ($cached) {
                return $cached;
            }
        }

        $dateFrom = $filters['date_from'] ?? date('Y-m-01');
        $dateTo = $filters['date_to'] ?? date('Y-m-t');
        $groupBy = $filters['group_by'] ?? 'month';

        $selectFields = "DATE_FORMAT(date, '%Y-%m') as period";
        $groupByClause = "DATE_FORMAT(date, '%Y-%m')";

        if ($groupBy === 'product') {
            $selectFields = "p.name as product_name, SUM(tblaccounts.amount) as total";
            $groupByClause = "p.id";
        } elseif ($groupBy === 'client') {
            $selectFields = "c.id as client_id, CONCAT(c.firstname, ' ', c.lastname) as client_name, SUM(tblaccounts.amount) as total";
            $groupByClause = "c.id";
        }

        $query = "SELECT {$selectFields},
                         SUM(tblaccounts.amount) as total,
                         COUNT(*) as transaction_count,
                         AVG(tblaccounts.amount) as avg_transaction
                  FROM tblaccounts
                  LEFT JOIN tblclients c ON tblaccounts.userid = c.id
                  LEFT JOIN tblinvoiceitems ii ON ii.invoiceid = tblaccounts.invoiceid
                  LEFT JOIN tblproducts p ON ii.relid = p.id
                  WHERE tblaccounts.date >= '{$dateFrom}'
                    AND tblaccounts.date <= '{$dateTo}'
                    AND tblaccounts.transkind = 1
                  GROUP BY {$groupByClause}
                  ORDER BY period DESC, total DESC";

        $results = full_query($query);
        $data = [];
        while ($row = mysql_fetch_assoc($results)) {
            $data[] = $row;
        }

        $summary = [
            'total_revenue' => array_sum(array_column($data, 'total')),
            'total_transactions' => array_sum(array_column($data, 'transaction_count')),
            'avg_transaction' => count($data) > 0 ? array_sum(array_column($data, 'total')) / count($data) : 0,
        ];

        $report = [
            'filters' => $filters,
            'summary' => $summary,
            'data' => $data,
            'generated_at' => date('Y-m-d H:i:s'),
        ];

        if ($this->cacheEnabled) {
            $this->saveToCache($cacheKey, $report);
        }

        return $report;
    }

    /**
     * Generate invoice aging report
     */
    public function generateAgingReport($filters = [])
    {
        $cacheKey = $this->generateCacheKey('aging', $filters);

        $agingBuckets = isset($filters['aging_buckets'])
            ? array_map('intval', explode(',', $filters['aging_buckets']))
            : [30, 60, 90];

        $query = "SELECT
                    i.id as invoice_id,
                    i.invoicenum,
                    c.id as client_id,
                    CONCAT(c.firstname, ' ', c.lastname) as client_name,
                    c.email as client_email,
                    i.total,
                    i.amount_paid,
                    (i.total - i.amount_paid) as balance,
                    i.duedate,
                    i.date,
                    DATEDIFF(NOW(), i.duedate) as days_overdue,
                    CASE
                        WHEN DATEDIFF(NOW(), i.duedate) <= 0 THEN 'current'
                        WHEN DATEDIFF(NOW(), i.duedate) <= " . $agingBuckets[0] . " THEN '1-" . $agingBuckets[0] . "'
                        WHEN DATEDIFF(NOW(), i.duedate) <= " . $agingBuckets[1] . " THEN '" . ($agingBuckets[0]+1) . "-" . $agingBuckets[1] . "'
                        WHEN DATEDIFF(NOW(), i.duedate) <= " . $agingBuckets[2] . " THEN '" . ($agingBuckets[1]+1) . "-" . $agingBuckets[2] . "'
                        ELSE '" . ($agingBuckets[2]+1) . "+'
                    END as aging_bucket
                   FROM tblinvoices i
                   JOIN tblclients c ON i.userid = c.id
                   WHERE i.status IN ('Unpaid', 'Overdue')
                   ORDER BY days_overdue DESC";

        $results = full_query($query);
        $data = [];
        while ($row = mysql_fetch_assoc($results)) {
            $data[] = $row;
        }

        // Calculate bucket totals
        $bucketTotals = [
            'current' => 0,
            '1-' . $agingBuckets[0] => 0,
            ($agingBuckets[0]+1) . '-' . $agingBuckets[1] => 0,
            ($agingBuckets[1]+1) . '-' . $agingBuckets[2] => 0,
            ($agingBuckets[2]+1) . '+' => 0,
        ];

        foreach ($data as $row) {
            $bucketTotals[$row['aging_bucket']] += $row['balance'];
        }

        return [
            'aging_buckets' => $agingBuckets,
            'bucket_totals' => $bucketTotals,
            'total_outstanding' => array_sum($bucketTotals),
            'invoices' => $data,
            'generated_at' => date('Y-m-d H:i:s'),
        ];
    }

    /**
     * Generate payment method report
     */
    public function generatePaymentMethodReport($filters = [])
    {
        $dateFrom = $filters['date_from'] ?? date('Y-m-01');
        $dateTo = $filters['date_to'] ?? date('Y-m-t');

        $query = "SELECT
                    gateway,
                    COUNT(*) as transaction_count,
                    SUM(amount) as total_amount,
                    AVG(amount) as avg_amount,
                    MIN(amount) as min_amount,
                    MAX(amount) as max_amount
                   FROM tblaccounts
                   WHERE date >= '{$dateFrom}'
                     AND date <= '{$dateTo}'
                     AND transkind = 1
                   GROUP BY gateway
                   ORDER BY total_amount DESC";

        $results = full_query($query);
        $data = [];
        while ($row = mysql_fetch_assoc($results)) {
            $data[] = $row;
        }

        $totalAmount = array_sum(array_column($data, 'total_amount'));

        // Add percentage
        foreach ($data as &$row) {
            $row['percentage'] = $totalAmount > 0
                ? round(($row['total_amount'] / $totalAmount) * 100, 2)
                : 0;
        }

        return [
            'filters' => $filters,
            'total_amount' => $totalAmount,
            'methods' => $data,
            'generated_at' => date('Y-m-d H:i:s'),
        ];
    }

    /**
     * Generate refund report
     */
    public function generateRefundReport($filters = [])
    {
        $dateFrom = $filters['date_from'] ?? date('Y-m-01');
        $dateTo = $filters['date_to'] ?? date('Y-m-t');

        // If using refund module
        $query = "SELECT
                    mr.id as refund_id,
                    mr.refund_number,
                    mr.requested_amount,
                    mr.refunded_amount,
                    mr.type,
                    mr.status,
                    mr.reason,
                    mr.requested_at,
                    mr.completed_at,
                    c.id as client_id,
                    CONCAT(c.firstname, ' ', c.lastname) as client_name,
                    i.id as invoice_id,
                    i.invoicenum
                   FROM mod_refunds mr
                   JOIN tblclients c ON mr.client_id = c.id
                   LEFT JOIN tblinvoices i ON mr.invoice_id = i.id
                   WHERE mr.requested_at >= '{$dateFrom}'
                     AND mr.requested_at <= '{$dateTo}'
                   ORDER BY mr.requested_at DESC";

        $results = full_query($query);
        $data = [];
        while ($row = mysql_fetch_assoc($results)) {
            $data[] = $row;
        }

        return [
            'filters' => $filters,
            'total_refunds' => count($data),
            'total_amount' => array_sum(array_column($data, 'refunded_amount')),
            'refunds' => $data,
            'generated_at' => date('Y-m-d H:i:s'),
        ];
    }

    /**
     * Generate client billing summary
     */
    public function generateClientBillingSummary($clientId)
    {
        $query = "SELECT
                    c.id as client_id,
                    CONCAT(c.firstname, ' ', c.lastname) as client_name,
                    c.email,
                    c.datecreated as client_since,
                    COUNT(DISTINCT i.id) as total_invoices,
                    SUM(i.total) as total_billed,
                    SUM(i.amount_paid) as total_paid,
                    (SUM(i.total) - SUM(i.amount_paid)) as outstanding_balance,
                    COUNT(DISTINCT CASE WHEN i.status = 'Unpaid' OR i.status = 'Overdue' THEN i.id END) as unpaid_invoices
                   FROM tblclients c
                   LEFT JOIN tblinvoices i ON c.id = i.userid
                   WHERE c.id = " . (int)$clientId . "
                   GROUP BY c.id";

        $result = full_query($query);
        $summary = mysql_fetch_assoc($result);

        // Get recent invoices
        $recentInvoices = full_query("SELECT * FROM tblinvoices WHERE userid = " . (int)$clientId . " ORDER BY date DESC LIMIT 10");
        $summary['recent_invoices'] = [];
        while ($row = mysql_fetch_assoc($recentInvoices)) {
            $summary['recent_invoices'][] = $row;
        }

        // Get payment history
        $paymentHistory = full_query("SELECT * FROM tblaccounts WHERE userid = " . (int)$clientId . " AND transkind = 1 ORDER BY date DESC LIMIT 10");
        $summary['recent_payments'] = [];
        while ($row = mysql_fetch_assoc($paymentHistory)) {
            $summary['recent_payments'][] = $row;
        }

        return $summary;
    }

    /**
     * Create saved report
     */
    public function createReport($data)
    {
        $id = Capsule::table('mod_billing_reports')->insertGetId([
            'name' => $data['name'],
            'description' => $data['description'] ?? null,
            'report_type' => $data['report_type'],
            'config' => json_encode($data['config'] ?? []),
            'columns' => json_encode($data['columns'] ?? []),
            'filters' => json_encode($data['filters'] ?? []),
            'sort_order' => $data['sort_order'] ?? null,
            'group_by' => $data['group_by'] ?? null,
            'chart_type' => $data['chart_type'] ?? null,
            'is_shared' => $data['is_shared'] ?? 0,
            'is_favorite' => $data['is_favorite'] ?? 0,
            'created_by' => $_SESSION['adminid'] ?? 1,
        ]);

        return ['success' => true, 'id' => $id];
    }

    /**
     * Get saved reports
     */
    public function getSavedReports($reportType = null)
    {
        $query = Capsule::table('mod_billing_reports')
            ->orderBy('is_favorite', 'desc')
            ->orderBy('name', 'asc');

        if ($reportType) {
            $query->where('report_type', $reportType);
        }

        return $query->get();
    }

    /**
     * Execute saved report
     */
    public function executeSavedReport($reportId)
    {
        $report = Capsule::table('mod_billing_reports')->find($reportId);
        if (!$report) {
            return ['success' => false, 'message' => 'Report not found'];
        }

        // Log execution
        $logId = Capsule::table('mod_report_logs')->insertGetId([
            'report_id' => $reportId,
            'status' => 'running',
            'executed_by' => $_SESSION['adminid'] ?? null,
        ]);

        $startTime = microtime(true);

        try {
            $config = json_decode($report->config, true);
            $filters = json_decode($report->filters, true);

            $data = match ($report->report_type) {
                'revenue' => $this->generateRevenueReport($filters),
                'aging' => $this->generateAgingReport($filters),
                'payments' => $this->generatePaymentMethodReport($filters),
                'refunds' => $this->generateRefundReport($filters),
                default => [],
            };

            $executionTime = round(microtime(true) - $startTime);

            Capsule::table('mod_report_logs')
                ->where('id', $logId)
                ->update([
                    'status' => 'completed',
                    'execution_time' => $executionTime,
                    'row_count' => count($data['data'] ?? $data['invoices'] ?? []),
                ]);

            return [
                'success' => true,
                'report' => $report,
                'data' => $data,
                'execution_time' => $executionTime,
            ];

        } catch (Exception $e) {
            Capsule::table('mod_report_logs')
                ->where('id', $logId)
                ->update([
                    'status' => 'failed',
                    'error_message' => $e->getMessage(),
                ]);

            return ['success' => false, 'message' => $e->getMessage()];
        }
    }

    /**
     * Export report to CSV
     */
    public function exportToCsv($reportId, $data)
    {
        $report = Capsule::table('mod_billing_reports')->find($reportId);

        header('Content-Type: text/csv');
        header('Content-Disposition: attachment; filename="' . $report->name . '_' . date('Y-m-d') . '.csv"');

        $output = fopen('php://output', 'w');

        if (!empty($data['data'])) {
            // Write header
            fputcsv($output, array_keys($data['data'][0]));

            // Write data
            foreach ($data['data'] as $row) {
                fputcsv($output, $row);
            }
        }

        fclose($output);
        exit;
    }

    /**
     * Cache helpers
     */
    protected function generateCacheKey($type, $filters)
    {
        return md5($type . serialize($filters));
    }

    protected function getFromCache($key)
    {
        $cached = Capsule::table('mod_report_cache')
            ->where('cache_key', $key)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->first();

        return $cached ? json_decode($cached->data, true) : null;
    }

    protected function saveToCache($key, $data)
    {
        Capsule::table('mod_report_cache')->insert([
            'report_id' => 0,
            'cache_key' => $key,
            'data' => json_encode($data),
            'expires_at' => date('Y-m-d H:i:s', strtotime("+{$this->cacheMinutes} minutes")),
        ]);
    }
}
```

## Hook Integrations

### DailyCronJob Hook - Process scheduled reports

```php
<?php
/**
 * Hook: DailyCronJob - Generate and send scheduled reports
 */
add_hook('DailyCronJob', 1, function($vars) {
    $schedules = Capsule::table('mod_report_schedules')
        ->where('is_active', 1)
        ->where('next_run', '<=', date('Y-m-d H:i:s'))
        ->get();

    foreach ($schedules as $schedule) {
        try {
            $reportService = new \WHMCS\Module\BillingReport\BillingReportService();
            $result = $reportService->executeSavedReport($schedule->report_id);

            if ($result['success']) {
                $report = $result['report'];
                $recipients = json_decode($schedule->recipients, true);

                foreach ($recipients as $email) {
                    // Generate report file
                    $file = $reportService->generateReportFile($report, $schedule->format);

                    // Send email with attachment
                    sendEmail('Scheduled Report', $email, $report->name, '', [
                        'report_name' => $report->name,
                        'generated_at' => date('Y-m-d H:i:s'),
                    ], $file);
                }

                // Update last run and calculate next run
                Capsule::table('mod_report_schedules')
                    ->where('id', $schedule->id)
                    ->update([
                        'last_run' => date('Y-m-d H:i:s'),
                        'next_run' => calculateNextRun($schedule->schedule_type, $schedule->send_day, $schedule->send_time),
                    ]);
            }

        } catch (Exception $e) {
            logActivity("Scheduled report failed: " . $e->getMessage());
        }
    }
});

function calculateNextRun($type, $day, $time)
{
    $sendTime = strtotime($time);
    $date = new DateTime();

    switch ($type) {
        case 'daily':
            $date->modify('+1 day');
            break;
        case 'weekly':
            $date->modify('+1 week');
            break;
        case 'monthly':
            $date->modify('+1 month');
            break;
        case 'quarterly':
            $date->modify('+3 months');
            break;
    }

    $date->setTime(date('H', $sendTime), date('i', $sendTime));
    return $date->format('Y-m-d H:i:s');
}
```

## Development Checklist

### Phase 1: Core Reporting System
- [ ] Create database tables
- [ ] Build BillingReportService class
- [ ] Implement report generation methods
- [ ] Add caching system
- [ ] Test report execution

### Phase 2: Report Types
- [ ] Revenue summary report
- [ ] Invoice aging report
- [ ] Payment methods breakdown
- [ ] Refund analysis
- [ ] Client billing summary

### Phase 3: Admin Interface
- [ ] Report dashboard
- [ ] Report builder
- [ ] Saved reports management
- [ ] Report execution
- [ ] Export functionality

### Phase 4: Export Features
- [ ] CSV export
- [ ] Excel export
- [ ] PDF generation
- [ ] Email with attachments
- [ ] Bulk export

### Phase 5: Scheduling
- [ ] Schedule creation
- [ ] Schedule management
- [ ] Email delivery
- [ ] Recurrence calculation
- [ ] Execution logging

### Phase 6: Charts & Visualization
- [ ] Bar charts
- [ ] Line charts
- [ ] Pie charts
- [ ] Data tables
- [ ] Trend analysis

### Phase 7: Caching & Performance
- [ ] Query optimization
- [ ] Result caching
- [ ] Background generation
- [ ] Large dataset handling
- [ ] Pagination

### Phase 8: Filters & Customization
- [ ] Date range filters
- [ ] Client filters
- [ ] Product filters
- [ ] Status filters
- [ ] Custom columns

### Phase 9: Security
- [ ] Access control
- [ ] Data visibility
- [ ] Export restrictions
- [ ] Audit logging
- [ ] Rate limiting

### Phase 10: Testing
- [ ] Report accuracy
- [ ] Export verification
- [ ] Scheduling reliability
- [ ] Performance testing
- [ ] Data integrity
