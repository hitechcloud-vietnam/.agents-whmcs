# WHMCS Report Automation Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for automating report generation in WHMCS including scheduled reports, exports, and custom analytics.

## When to Use

- Setting up automated report generation
- Creating custom report templates
- Exporting data in various formats
- Building analytics dashboards

## Report Automation Patterns

### 1. Report Generator Class

```php
<?php
namespace WHMCS\Reports;

class AutomatedReportGenerator {
    private string $reportPath;
    private array $supportedFormats = ['pdf', 'csv', 'excel', 'json'];

    public function generateReport(string $reportType, string $format, array $params = []): array {
        if (!in_array($format, $this->supportedFormats)) {
            throw new \InvalidArgumentException("Unsupported format: $format");
        }

        $data = $this->fetchReportData($reportType, $params);
        $filename = $this->generateFilename($reportType, $format);

        switch ($format) {
            case 'pdf':
                return $this->generatePDF($reportType, $data, $filename);
            case 'csv':
                return $this->generateCSV($data, $filename);
            case 'excel':
                return $this->generateExcel($data, $filename);
            case 'json':
                return $this->generateJSON($data, $filename);
            default:
                throw new \InvalidArgumentException("Format not implemented: $format");
        }
    }

    private function fetchReportData(string $reportType, array $params): array {
        $className = 'WHMCS\\Reports\\ReportType\\' . ucfirst($reportType) . 'Report';

        if (!class_exists($className)) {
            throw new \InvalidArgumentException("Unknown report type: $reportType");
        }

        $report = new $className();
        return $report->fetch($params);
    }

    private function generateFilename(string $reportType, string $format): string {
        $timestamp = date('Y-m-d_His');
        return "{$reportType}_report_{$timestamp}.{$format}";
    }

    private function generatePDF(string $reportType, array $data, string $filename): array {
        $pdf = new \TCPDF();
        $pdf->SetCreator('WHMCS Report Generator');
        $pdf->SetAuthor('WHMCS');
        $pdf->SetTitle(ucfirst($reportType) . ' Report');
        $pdf->SetMargins(15, 15, 15);

        $pdf->AddPage();
        $pdf->SetFont('helvetica', 'B', 16);
        $pdf->Cell(0, 10, ucfirst($reportType) . ' Report', 0, 1, 'C');
        $pdf->SetFont('helvetica', '', 10);
        $pdf->Cell(0, 8, 'Generated: ' . date('Y-m-d H:i:s'), 0, 1, 'C');
        $pdf->Ln(10);

        // Generate content based on data
        $pdf->WriteHTML($this->renderHTMLTable($data));
        $pdf->Output($this->reportPath . $filename, 'F');

        return ['path' => $this->reportPath . $filename, 'filename' => $filename];
    }

    private function generateCSV(array $data, string $filename): array {
        $handle = fopen($this->reportPath . $filename, 'w');

        if (empty($data)) {
            fclose($handle);
            return ['path' => $this->reportPath . $filename, 'filename' => $filename];
        }

        // Headers
        fputcsv($handle, array_keys($data[0]));

        // Data rows
        foreach ($data as $row) {
            fputcsv($handle, array_values($row));
        }

        fclose($handle);

        return ['path' => $this->reportPath . $filename, 'filename' => $filename];
    }

    private function generateExcel(array $data, string $filename): array {
        require_once __DIR__ . '/../vendor/phpoffice/phpexcel/Classes/PHPExcel.php';

        $excel = new \PHPExcel();
        $excel->setActiveSheetIndex(0);
        $sheet = $excel->getActiveSheet();

        if (empty($data)) {
            $sheet->setCellValue('A1', 'No data available');
            $excel->setActiveSheetIndex(0);
            $writer = \PHPExcel_IOFactory::createWriter($excel, 'Excel2007');
            $writer->save($this->reportPath . $filename);

            return ['path' => $this->reportPath . $filename, 'filename' => $filename];
        }

        // Headers
        $col = 'A';
        foreach (array_keys($data[0]) as $header) {
            $sheet->setCellValue($col . '1', $header);
            $col++;
        }

        // Data rows
        $row = 2;
        foreach ($data as $record) {
            $col = 'A';
            foreach ($record as $value) {
                $sheet->setCellValue($col . $row, $value);
                $col++;
            }
            $row++;
        }

        $writer = \PHPExcel_IOFactory::createWriter($excel, 'Excel2007');
        $writer->save($this->reportPath . $filename);

        return ['path' => $this->reportPath . $filename, 'filename' => $filename];
    }

    private function generateJSON(array $data, string $filename): array {
        $json = json_encode(['generated' => date('Y-m-d H:i:s'), 'count' => count($data), 'data' => $data], JSON_PRETTY_PRINT);
        file_put_contents($this->reportPath . $filename, $json);

        return ['path' => $this->reportPath . $filename, 'filename' => $filename];
    }

    private function renderHTMLTable(array $data): string {
        if (empty($data)) {
            return '<p>No data available</p>';
        }

        $html = '<table border="1" cellpadding="5">';
        $html .= '<tr>';
        foreach (array_keys($data[0]) as $header) {
            $html .= '<th>' . htmlspecialchars($header) . '</th>';
        }
        $html .= '</tr>';

        foreach ($data as $row) {
            $html .= '<tr>';
            foreach ($row as $value) {
                $html .= '<td>' . htmlspecialchars((string)$value) . '</td>';
            }
            $html .= '</tr>';
        }

        $html .= '</table>';
        return $html;
    }
}
```

### 2. Report Types

```php
<?php
namespace WHMCS\Reports\ReportType;

class RevenueReport {
    public function fetch(array $params): array {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');

        $invoices = Capsule::table('tblinvoices')
            ->selectRaw('DATE(invoicedate) as date, COUNT(*) as count, SUM(total) as amount')
            ->where('invoicedate', '>=', $startDate)
            ->where('invoicedate', '<=', $endDate)
            ->where('status', 'Paid')
            ->groupBy('date')
            ->orderBy('date')
            ->get()
            ->toArray();

        return array_map(function($row) {
            return [
                'Date' => $row->date,
                'Invoices Count' => $row->count,
                'Total Amount' => number_format($row->amount, 2),
            ];
        }, $invoices);
    }
}

class ClientActivityReport {
    public function fetch(array $params): array {
        $limit = $params['limit'] ?? 100;

        $clients = Capsule::table('tblclients')
            ->selectRaw('*, COUNT(tblhosting.id) as services_count')
            ->selectRaw('SUM(tblhosting.amount) as total_spent')
            ->leftJoin('tblhosting', 'tblhosting.userid', '=', 'tblclients.id')
            ->where('tblhosting.domainstatus', 'Active')
            ->groupBy('tblclients.id')
            ->orderBy('total_spent', 'desc')
            ->limit($limit)
            ->get()
            ->toArray();

        return array_map(function($client) {
            return [
                'Client ID' => $client->id,
                'Name' => $client->firstname . ' ' . $client->lastname,
                'Email' => $client->email,
                'Services' => $client->services_count,
                'Total Spent' => number_format($client->total_spent ?? 0, 2),
                'Joined' => $client->datecreated,
            ];
        }, $clients);
    }
}

class ServiceHealthReport {
    public function fetch(array $params): array {
        $servers = Capsule::table('tblservers')
            ->selectRaw('tblservers.*')
            ->selectRaw('COUNT(tblhosting.id) as active_services')
            ->leftJoin('tblhosting', function($join) {
                $join->on('tblhosting.server', '=', 'tblservers.id')
                     ->where('tblhosting.domainstatus', 'Active');
            })
            ->groupBy('tblservers.id')
            ->get()
            ->toArray();

        return array_map(function($server) {
            return [
                'Server' => $server->name,
                'Hostname' => $server->hostname,
                'Active Services' => $server->active_services,
                'Status' => $server->disabled ? 'Disabled' : 'Active',
            ];
        }, $servers);
    }
}
```

### 3. Scheduled Report Configuration

```php
<?php
namespace WHMCS\Reports;

class ReportScheduler {
    public function scheduleReport(array $schedule): int {
        $id = Capsule::table('mod_scheduled_reports')->insertGetId([
            'name' => $schedule['name'],
            'report_type' => $schedule['report_type'],
            'format' => $schedule['format'],
            'recipients' => json_encode($schedule['recipients']),
            'schedule_cron' => $schedule['cron_expression'],
            'params' => json_encode($schedule['params'] ?? []),
            'last_run' => null,
            'next_run' => $this->calculateNextRun($schedule['cron_expression']),
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        return $id;
    }

    public function getDueReports(): array {
        $now = date('Y-m-d H:i:s');

        return Capsule::table('mod_scheduled_reports')
            ->where('next_run', '<=', $now)
            ->where('active', 1)
            ->get()
            ->toArray();
    }

    public function processScheduledReports(): void {
        $dueReports = $this->getDueReports();
        $generator = new AutomatedReportGenerator();

        foreach ($dueReports as $report) {
            try {
                $result = $generator->generateReport(
                    $report->report_type,
                    $report->format,
                    json_decode($report->params, true)
                );

                // Send to recipients
                $this->sendReport(
                    json_decode($report->recipients, true),
                    $result,
                    $report->name
                );

                // Update last run time
                Capsule::table('mod_scheduled_reports')
                    ->where('id', $report->id)
                    ->update([
                        'last_run' => date('Y-m-d H:i:s'),
                        'next_run' => $this->calculateNextRun($report->schedule_cron),
                        'last_status' => 'success',
                    ]);

            } catch (\Exception $e) {
                Capsule::table('mod_scheduled_reports')
                    ->where('id', $report->id)
                    ->update([
                        'last_run' => date('Y-m-d H:i:s'),
                        'last_status' => 'error',
                        'last_error' => $e->getMessage(),
                    ]);

                logActivity("Report generation failed: " . $e->getMessage());
            }
        }
    }

    private function calculateNextRun(string $cronExpression): string {
        // Parse cron expression and calculate next run time
        // This is a simplified implementation
        $parts = explode(' ', $cronExpression);

        if (count($parts) !== 5) {
            return date('Y-m-d H:i:s', strtotime('+1 day'));
        }

        // For simplicity, calculate based on period
        $minute = $parts[0];
        $hour = $parts[1];

        if ($minute === '*' && $hour === '*') {
            return date('Y-m-d H:i:s', strtotime('+1 hour'));
        }

        if ($minute === '*') {
            $next = strtotime("+1 day midnight");
            $next = strtotime("+1 day", strtotime(date('Y-m-d') . " {$hour}:{$minute}:00"));
        } else {
            $next = strtotime("+1 hour");
        }

        return date('Y-m-d H:i:s', $next);
    }

    private function sendReport(array $recipients, array $reportFile, string $reportName): void {
        $mailer = new WHMCS_Mail();
        $mailer->Subject = "Scheduled Report: $reportName";
        $mailer->Body = "Please find attached the scheduled report: $reportName";

        foreach ($recipients as $email) {
            $mailer->addAddress($email);
        }

        $mailer->addAttachment($reportFile['path']);
        $mailer->send();
    }
}
```

### 4. Report Hooks

```php
<?php
// hooks.php
add_hook('DailyCronJob', 1, function($vars) {
    $scheduler = new ReportScheduler();
    $scheduler->processScheduledReports();
});

add_hook('ReportGenerationComplete', 1, function($vars) {
    // Log report generation
    Capsule::table('mod_report_logs')->insert([
        'report_type' => $vars['report_type'],
        'format' => $vars['format'],
        'filename' => $vars['filename'],
        'generated_at' => date('Y-m-d H:i:s'),
        'user_id' => $_SESSION['adminid'] ?? null,
    ]);
});
```

### 5. Database Schema

```php
<?php
function createReportTables(): void {
    Capsule::schema()->create('mod_scheduled_reports', function($t) {
        $t->increments('id');
        $t->string('name');
        $t->string('report_type');
        $t->string('format');
        $t->text('recipients'); // JSON array of emails
        $t->string('schedule_cron'); // Cron expression
        $t->text('params'); // JSON for custom parameters
        $t->timestamp('last_run')->nullable();
        $t->timestamp('next_run')->nullable();
        $t->string('last_status')->nullable();
        $t->text('last_error')->nullable();
        $t->boolean('active')->default(1);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_report_logs', function($t) {
        $t->increments('id');
        $t->string('report_type');
        $t->string('format');
        $t->string('filename');
        $t->integer('user_id')->unsigned()->nullable();
        $t->timestamp('generated_at');
        $t->integer('rows_count')->unsigned()->default(0);
        $t->bigInteger('file_size')->unsigned()->default(0);
    });
}
```

## Checklist

- [ ] Report generator class
- [ ] Report type implementations
- [ ] Multiple format support (PDF, CSV, Excel, JSON)
- [ ] Scheduled report configuration
- [ ] Cron job integration
- [ ] Email delivery to recipients
- [ ] Report logging
- [ ] Admin interface for report management

---

**Related Skills:**
- whmcs-cron-automation
- whmcs-reporting
- whmcs-metrics-analytics
- whmcs-data-export
