# WHMCS Data Export Workflow

## Overview
This workflow handles data export operations for reporting, compliance, and migration purposes.

## Step 1: Export Service

```php
<?php
// src/Service/ExportService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class ExportService
{
    public function exportClients(array $filters = [], string $format = 'csv'): array
    {
        $query = Capsule::table('tblclients');

        if (!empty($filters['status'])) {
            $query->where('status', $filters['status']);
        }

        if (!empty($filters['date_from'])) {
            $query->where('datecreated', '>=', $filters['date_from']);
        }

        if (!empty($filters['date_to'])) {
            $query->where('datecreated', '<=', $filters['date_to']);
        }

        $clients = $query->get();

        return match ($format) {
            'csv' => $this->toCsv($clients->toArray()),
            'json' => json_encode($clients),
            'xlsx' => $this->toExcel($clients->toArray()),
            default => ['success' => false, 'error' => 'Unknown format']
        };
    }

    public function exportInvoices(array $filters = [], string $format = 'csv'): array
    {
        $query = Capsule::table('tblinvoices');

        if (!empty($filters['status'])) {
            $query->where('status', $filters['status']);
        }

        if (!empty($filters['date_from'])) {
            $query->where('date', '>=', $filters['date_from']);
        }

        $invoices = $query->get();

        return match ($format) {
            'csv' => $this->toCsv($invoices->toArray()),
            'json' => json_encode($invoices),
            default => ['success' => false, 'error' => 'Unknown format']
        };
    }

    public function exportServices(array $filters = [], string $format = 'csv'): array
    {
        $query = Capsule::table('tblhosting')
            ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->select('tblhosting.*', 'tblclients.email', 'tblproducts.name as product_name');

        if (!empty($filters['status'])) {
            $query->where('tblhosting.domainstatus', $filters['status']);
        }

        $services = $query->get();

        return match ($format) {
            'csv' => $this->toCsv($services->toArray()),
            'json' => json_encode($services),
            default => ['success' => false, 'error' => 'Unknown format']
        };
    }

    private function toCsv(array $data): string
    {
        if (empty($data)) {
            return '';
        }

        $output = fopen('php://temp', 'r+');

        // Headers
        fputcsv($output, array_keys($data[0]));

        // Data
        foreach ($data as $row) {
            fputcsv($output, (array)$row);
        }

        rewind($output);
        $csv = stream_get_contents($output);
        fclose($output);

        return $csv;
    }

    private function toExcel(array $data): string
    {
        // For Excel, you'd typically use a library like PhpSpreadsheet
        // This is a placeholder that returns JSON
        return json_encode($data);
    }
}
```

## Step 2: Scheduled Export Cron

```php
<?php
// includes/cron/scheduled_export_cron.php

require_once __DIR__ . '/../../init.php';

use WHMCS\Module\Addon\YourModule\Service\ExportService;

$service = new ExportService();

// Export monthly report
$report = $service->exportInvoices([
    'date_from' => date('Y-m-01'),
    'date_to' => date('Y-m-t')
], 'csv');

// Save to storage
file_put_contents(
    dirname(__DIR__, 3) . '/storage/exports/invoice_report_' . date('Y-m') . '.csv',
    $report
);
```

## Verification Checklist

- [ ] Export service implemented
- [ ] CSV export working
- [ ] JSON export working
- [ ] Filters working correctly
- [ ] Large dataset handling tested
