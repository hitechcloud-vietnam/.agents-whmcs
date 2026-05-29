# WHMCS Export Automation Workflow

## Overview
This workflow implements automated data export capabilities for WHMCS.

## Prerequisites
- WHMCS with database access
- Storage for exported files
- Optional: FTP/S3 credentials for remote storage

## Step-by-Step Process

### Step 1: Create Export Manager
```php
<?php
// /includes/export/ExportManager.php

namespace WHMCS\Export;

class ExportManager
{
    private $exportDir;

    public function __construct()
    {
        $this->exportDir = getConfig('export_directory') ?? __DIR__ . '/../../exports';
        if (!is_dir($this->exportDir)) {
            mkdir($this->exportDir, 0755, true);
        }
    }

    /**
     * Export to CSV
     */
    public function exportToCSV(string $type, array $filters = [], array $options = []): string
    {
        $data = $this->getExportData($type, $filters);
        $filename = $this->exportDir . '/' . $type . '_' . date('Y-m-d_His') . '.csv';

        $handle = fopen($filename, 'w');

        if (!empty($data)) {
            // Write header
            fputcsv($handle, array_keys((array)$data[0]));

            // Write data
            foreach ($data as $row) {
                fputcsv($handle, (array)$row);
            }
        }

        fclose($handle);

        // Upload to remote if configured
        if ($options['upload_remote'] ?? false) {
            $this->uploadRemote($filename);
        }

        return $filename;
    }

    /**
     * Export to JSON
     */
    public function exportToJSON(string $type, array $filters = []): string
    {
        $data = $this->getExportData($type, $filters);
        $filename = $this->exportDir . '/' . $type . '_' . date('Y-m-d_His') . '.json';

        file_put_contents($filename, json_encode($data, JSON_PRETTY_PRINT));

        return $filename;
    }

    /**
     * Export to XML
     */
    public function exportToXML(string $type, array $filters = []): string
    {
        $data = $this->getExportData($type, $filters);
        $filename = $this->exportDir . '/' . $type . '_' . date('Y-m-d_His') . '.xml';

        $xml = new SimpleXMLElement('<?xml version="1.0" encoding="UTF-8"?><root/>');

        foreach ($data as $item) {
            $itemXml = $xml->addChild($type);
            foreach ((array)$item as $key => $value) {
                $itemXml->addChild($key, htmlspecialchars($value));
            }
        }

        $xml->asXML($filename);

        return $filename;
    }

    /**
     * Get data for export
     */
    private function getExportData(string $type, array $filters): array
    {
        return match ($type) {
            'clients' => $this->getClientsData($filters),
            'services' => $this->getServicesData($filters),
            'invoices' => $this->getInvoicesData($filters),
            'domains' => $this->getDomainsData($filters),
            'transactions' => $this->getTransactionsData($filters),
            'tickets' => $this->getTicketsData($filters),
            default => []
        };
    }

    private function getClientsData(array $filters): array
    {
        return Capsule::table('tblclients')
            ->when(isset($filters['status']), fn($q) => $q->where('status', $filters['status']))
            ->when(isset($filters['created_after']), fn($q) => $q->where('datecreated', '>=', $filters['created_after']))
            ->get()
            ->toArray();
    }

    private function getServicesData(array $filters): array
    {
        return Capsule::table('tblhosting')
            ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->select('tblhosting.*', 'tblclients.email', 'tblproducts.name as product_name')
            ->when(isset($filters['status']), fn($q) => $q->where('tblhosting.domainstatus', $filters['status']))
            ->get()
            ->toArray();
    }

    private function getInvoicesData(array $filters): array
    {
        return Capsule::table('tblinvoices')
            ->when(isset($filters['status']), fn($q) => $q->where('status', $filters['status']))
            ->when(isset($filters['date_from']), fn($q) => $q->where('date', '>=', $filters['date_from']))
            ->when(isset($filters['date_to']), fn($q) => $q->where('date', '<=', $filters['date_to']))
            ->get()
            ->toArray();
    }
}
```

### Step 2: Create Scheduled Export
```php
<?php
// /includes/hooks/export_hooks.php

add_hook('DailyCronJob', 1, function($vars) {
    $exportManager = new ExportManager();

    // Daily export of yesterday's invoices
    $yesterday = date('Y-m-d', strtotime('-1 day'));

    $filename = $exportManager->exportToCSV('invoices', [
        'date_from' => $yesterday,
        'date_to' => $yesterday,
        'status' => 'Paid'
    ], ['upload_remote' => true]);

    logActivity("Daily invoice export: {$filename}");

    return $filename;
});

add_hook('MonthlyCronJob', 1, function($vars) {
    $exportManager = new ExportManager();

    // Monthly full export
    $exportManager->exportToCSV('clients');
    $exportManager->exportToCSV('services');
    $exportManager->exportToCSV('invoices');
    $exportManager->exportToCSV('domains');

    return "Monthly exports completed";
});
```

## Related Workflows
- [WHMCS Import Automation](./whmcs-import-automation.md)
- [WHMCS Data Export Import](./whmcs-data-export-import.md)