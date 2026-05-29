# WHMCS Import Automation Workflow

## Overview
This workflow implements automated data import from external sources into WHMCS.

## Prerequisites
- WHMCS installation
- Source data in supported format (CSV, XML, API)
- PHP file handling capabilities

## Step-by-Step Process

### Step 1: Create Import Manager
```php
<?php
// /includes/import/ImportManager.php

namespace WHMCS\Import;

class ImportManager
{
    private $importLog = [];

    /**
     * Import from CSV
     */
    public function importFromCSV(string $filePath, string $type, array $options = []): array
    {
        $results = [
            'total' => 0,
            'imported' => 0,
            'skipped' => 0,
            'errors' => 0,
            'errors_log' => []
        ];

        $handle = fopen($filePath, 'r');

        if ($options['has_header'] ?? true) {
            fgetcsv($handle);
        }

        while (($row = fgetcsv($handle)) !== false) {
            $results['total']++;

            try {
                $this->importRow($type, $row, $options);
                $results['imported']++;
            } catch (Exception $e) {
                $results['errors']++;
                $results['errors_log'][] = [
                    'row' => $results['total'],
                    'error' => $e->getMessage(),
                    'data' => $row
                ];
            }
        }

        fclose($handle);

        $this->logImport($type, $results);

        return $results;
    }

    /**
     * Import from API
     */
    public function importFromAPI(string $endpoint, string $type, array $options = []): array
    {
        $results = [
            'imported' => 0,
            'errors' => 0
        ];

        $data = $this->fetchFromAPI($endpoint);

        foreach ($data as $item) {
            try {
                $this->importItem($type, $item, $options);
                $results['imported']++;
            } catch (Exception $e) {
                $results['errors']++;
                logActivity("API import error: " . $e->getMessage());
            }
        }

        return $results;
    }

    /**
     * Import row based on type
     */
    private function importRow(string $type, array $row, array $options): void
    {
        switch ($type) {
            case 'clients':
                $this->importClient($row, $options);
                break;
            case 'services':
                $this->importService($row, $options);
                break;
            case 'domains':
                $this->importDomain($row, $options);
                break;
            default:
                throw new Exception("Unknown import type: {$type}");
        }
    }

    /**
     * Import client
     */
    private function importClient(array $row, array $options): int
    {
        // Map columns
        $mapping = $options['mapping'] ?? [
            'firstname' => 0,
            'lastname' => 1,
            'email' => 2,
            'companyname' => 3,
            'phonenumber' => 4
        ];

        $data = [];
        foreach ($mapping as $field => $index) {
            $data[$field] = $row[$index] ?? null;
        }

        // Check if client exists
        $exists = Capsule::table('tblclients')
            ->where('email', $data['email'])
            ->exists();

        if ($exists && ($options['skip_existing'] ?? true)) {
            throw new Exception("Client already exists: {$data['email']}");
        }

        if ($exists) {
            // Update existing
            Capsule::table('tblclients')
                ->where('email', $data['email'])
                ->update($data);

            return Capsule::table('tblclients')
                ->where('email', $data['email'])
                ->value('id');
        }

        // Create new
        return Capsule::table('tblclients')->insertGetId($data);
    }

    /**
     * Validate import data
     */
    public function validateImport(string $filePath, string $type): array
    {
        $issues = [];
        $handle = fopen($filePath, 'r');

        $rowNum = 0;
        while (($row = fgetcsv($handle)) !== false) {
            $rowNum++;

            foreach ($row as $colNum => $value) {
                if (empty($value) && $this->isRequired($type, $colNum)) {
                    $issues[] = "Row {$rowNum}: Required field is empty";
                }
            }
        }

        fclose($handle);

        return [
            'valid' => empty($issues),
            'issues' => $issues,
            'total_rows' => $rowNum
        ];
    }

    private function isRequired(string $type, int $col): bool
    {
        $requiredCols = [
            'clients' => [0, 1, 2], // firstname, lastname, email
            'services' => [0, 1, 2], // client_email, product_id, domain
            'domains' => [0, 1]      // domain, registration_date
        ];

        return in_array($col, $requiredCols[$type] ?? []);
    }

    private function logImport(string $type, array $results)
    {
        Capsule::table('mod_import_log')->insert([
            'type' => $type,
            'total' => $results['total'],
            'imported' => $results['imported'],
            'skipped' => $results['skipped'],
            'errors' => $results['errors'],
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

### Step 2: Create Scheduled Import
```php
<?php
// /includes/hooks/import_hooks.php

add_hook('DailyCronJob', 1, function($vars) {
    $importManager = new ImportManager();

    // Import clients from FTP
    $remoteFile = 'ftp://user:pass@server.com/data/clients.csv';

    $results = $importManager->importFromCSV($remoteFile, 'clients', [
        'has_header' => true,
        'mapping' => [
            'firstname' => 0,
            'lastname' => 1,
            'email' => 2,
            'companyname' => 3
        ],
        'skip_existing' => true
    ]);

    return $results;
});
```

## Related Workflows
- [WHMCS Export Automation](./whmcs-export-automation.md)
- [WHMCS Data Export Import](./whmcs-data-export-import.md)