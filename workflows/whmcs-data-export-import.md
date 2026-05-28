# WHMCS Data Export/Import Workflow
# Version: 1.0 | Created: 2026-05-28

## Purpose

Comprehensive guide to exporting and importing data in WHMCS including client data, service information, billing records, custom data migration, and automated backup procedures.

## Prerequisites

- WHMCS installation with full admin access
- Database backup capability
- PHP memory limits appropriate for large datasets
- Migration tools or custom export scripts

## Workflow Steps

### Step 1: Prepare Export Data

Create comprehensive data export scripts:

```php
// modules/addons/data_manager/export.php

use WHMCS\Database\Capsule;

class DataExporter
{
    private string $exportPath;
    private array $exportConfig;

    public function __construct(string $exportPath = null)
    {
        $this->exportPath = $exportPath ?? ROOTDIR . '/storage/exports/';
        $this->exportConfig = [
            'include_archived' => false,
            'date_range'       => null,
            'compress'         => true,
        ];
    }

    /**
     * Export all client data
     */
    public function exportClients(array $options = []): array
    {
        $query = Capsule::table('tblclients')->where('status', '!=', 'Inactive');

        if (isset($options['date_from'])) {
            $query->where('datecreated', '>=', $options['date_from']);
        }

        if (isset($options['date_to'])) {
            $query->where('datecreated', '<=', $options['date_to']);
        }

        $clients = $query->get();

        $exportData = [];
        foreach ($clients as $client) {
            $exportData[] = [
                'id'           => $client->id,
                'email'        => $client->email,
                'companyname'  => $client->companyname,
                'firstname'    => $client->firstname,
                'lastname'     => $client->lastname,
                'address1'     => $client->address1,
                'address2'     => $client->address2,
                'city'         => $client->city,
                'state'        => $client->state,
                'postcode'     => $client->postcode,
                'country'      => $client->country,
                'phonenumber'  => $client->phonenumber,
                'datecreated'  => $client->datecreated,
                'status'       => $client->status,
            ];
        }

        $filename = 'clients_export_' . date('Y-m-d_His') . '.csv';
        $this->writeCsv($filename, $exportData);

        return ['filename' => $filename, 'count' => count($exportData)];
    }

    /**
     * Export services with related data
     */
    public function exportServices(array $options = []): array
    {
        $services = Capsule::table('tblhosting')
            ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->join('tblservers', 'tblhosting.server', '=', 'tblservers.id')
            ->select(
                'tblhosting.*',
                'tblclients.email as client_email',
                'tblclients.companyname as client_company',
                'tblproducts.name as product_name',
                'tblproducts.type as product_type',
                'tblservers.name as server_name'
            )
            ->get();

        $exportData = [];
        foreach ($services as $service) {
            $exportData[] = [
                'service_id'       => $service->id,
                'client_id'        => $service->userid,
                'client_email'     => $service->client_email,
                'domain'           => $service->domain,
                'product'          => $service->product_name,
                'product_type'     => $service->product_type,
                'server'           => $service->server_name,
                'regdate'          => $service->regdate,
                'nextduedate'      => $service->nextduedate,
                'status'           => $service->domainstatus,
                'billing_cycle'    => $service->billingcycle,
                'amount'           => $service->amount,
            ];
        }

        $filename = 'services_export_' . date('Y-m-d_His') . '.csv';
        $this->writeCsv($filename, $exportData);

        return ['filename' => $filename, 'count' => count($exportData)];
    }

    /**
     * Export invoices
     */
    public function exportInvoices(array $options = []): array
    {
        $query = Capsule::table('tblinvoices');

        if (isset($options['status'])) {
            $query->where('status', $options['status']);
        }

        if (isset($options['date_from'])) {
            $query->where('date', '>=', $options['date_from']);
        }

        $invoices = $query->get();

        // Get invoice items
        $exportData = [];
        foreach ($invoices as $invoice) {
            $items = Capsule::table('tblinvoiceitems')
                ->where('invoiceid', $invoice->id)
                ->get();

            $itemDescriptions = [];
            foreach ($items as $item) {
                $itemDescriptions[] = $item->description . ' (' . $item->amount . ')';
            }

            $exportData[] = [
                'invoice_id'    => $invoice->id,
                'user_id'       => $invoice->userid,
                'date'          => $invoice->date,
                'duedate'       => $invoice->duedate,
                'subtotal'      => $invoice->subtotal,
                'tax'           => $invoice->tax,
                'total'         => $invoice->total,
                'status'        => $invoice->status,
                'payment_method'=> $invoice->paymentmethod,
                'items'         => implode('; ', $itemDescriptions),
            ];
        }

        $filename = 'invoices_export_' . date('Y-m-d_His') . '.csv';
        $this->writeCsv($filename, $exportData);

        return ['filename' => $filename, 'count' => count($exportData)];
    }

    /**
     * Export transactions
     */
    public function exportTransactions(array $options = []): array
    {
        $query = Capsule::table('tblaccounts');

        if (isset($options['date_from'])) {
            $query->where('date', '>=', $options['date_from']);
        }
        if (isset($options['date_to'])) {
            $query->where('date', '<=', $options['date_to']);
        }

        $transactions = $query->orderBy('date', 'DESC')->get();

        $exportData = [];
        foreach ($transactions as $txn) {
            $exportData[] = [
                'id'           => $txn->id,
                'user_id'      => $txn->userid,
                'invoice_id'   => $txn->invoiceid,
                'date'         => $txn->date,
                'description'  => $txn->description,
                'amount'       => $txn->amount,
                'fees'         => $txn->fees,
                'gateway'      => $txn->gateway,
            ];
        }

        $filename = 'transactions_export_' . date('Y-m-d_His') . '.csv';
        $this->writeCsv($filename, $exportData);

        return ['filename' => $filename, 'count' => count($exportData)];
    }

    /**
     * Export domains
     */
    public function exportDomains(): array
    {
        $domains = Capsule::table('tbldomains')
            ->join('tblclients', 'tbldomains.userid', '=', 'tblclients.id')
            ->select('tbldomains.*', 'tblclients.email as client_email')
            ->get();

        $exportData = [];
        foreach ($domains as $domain) {
            $exportData[] = [
                'domain_id'       => $domain->id,
                'client_id'       => $domain->userid,
                'client_email'    => $domain->client_email,
                'domain'          => $domain->domain,
                'registration_date' => $domain->registrationdate,
                'expiry_date'     => $domain->expirydate,
                'next_due_date'   => $domain->nextduedate,
                'status'          => $domain->status,
                'dns_management' => $domain->dnsmanagement,
                'email_forwarding'=> $domain->emailforwarding,
                'id_protection'   => $domain->idprotection,
                'registrar'       => $domain->registrar,
            ];
        }

        $filename = 'domains_export_' . date('Y-m-d_His') . '.csv';
        $this->writeCsv($filename, $exportData);

        return ['filename' => $filename, 'count' => count($exportData)];
    }

    private function writeCsv(string $filename, array $data): void
    {
        if (empty($data)) {
            return;
        }

        $filepath = $this->exportPath . $filename;
        $handle = fopen($filepath, 'w');

        // Write header
        fputcsv($handle, array_keys($data[0]));

        // Write data
        foreach ($data as $row) {
            fputcsv($handle, $row);
        }

        fclose($handle);

        // Compress if enabled
        if ($this->exportConfig['compress']) {
            $this->compressFile($filepath);
        }
    }

    private function compressFile(string $filepath): void
    {
        $zipPath = $filepath . '.zip';
        $zip = new ZipArchive();
        $zip->open($zipPath, ZipArchive::CREATE);
        $zip->addFile($filepath, basename($filepath));
        $zip->close();
        unlink($filepath);
    }
}
```

### Step 2: Implement Data Import Procedures

Create robust import mechanisms:

```php
// modules/addons/data_manager/import.php

class DataImporter
{
    private array $errors = [];
    private array $warnings = [];
    private int $importedCount = 0;
    private int $skippedCount = 0;

    /**
     * Import clients from CSV
     */
    public function importClients(string $filepath): array
    {
        $handle = fopen($filepath, 'r');
        $headers = fgetcsv($handle);

        $requiredFields = ['email', 'firstname', 'lastname'];
        $this->validateHeaders($headers, $requiredFields);

        if (!empty($this->errors)) {
            return $this->getResult();
        }

        $lineCount = 1;
        while (($row = fgetcsv($handle)) !== false) {
            $lineCount++;
            $data = array_combine($headers, $row);

            try {
                $this->importClient($data);
                $this->importedCount++;
            } catch (Exception $e) {
                $this->errors[] = "Line {$lineCount}: " . $e->getMessage();
            }
        }

        fclose($handle);
        return $this->getResult();
    }

    private function importClient(array $data): void
    {
        // Check if client already exists
        $existing = Capsule::table('tblclients')
            ->where('email', $data['email'])
            ->first();

        if ($existing) {
            $this->warnings[] = "Skipped existing client: {$data['email']}";
            $this->skippedCount++;
            return;
        }

        // Validate required fields
        if (empty($data['firstname']) || empty($data['lastname']) || empty($data['email'])) {
            throw new Exception('Missing required fields');
        }

        // Create new client
        $clientId = Capsule::table('tblclients')->insertGetId([
            'email'       => $data['email'],
            'firstname'   => $data['firstname'],
            'lastname'    => $data['lastname'],
            'companyname' => $data['companyname'] ?? '',
            'address1'    => $data['address1'] ?? '',
            'address2'    => $data['address2'] ?? '',
            'city'        => $data['city'] ?? '',
            'state'       => $data['state'] ?? '',
            'postcode'    => $data['postcode'] ?? '',
            'country'     => $data['country'] ?? 'US',
            'phonenumber' => $data['phonenumber'] ?? '',
            'datecreated' => date('Y-m-d H:i:s'),
            'status'      => 'Active',
        ]);

        logActivity("Imported client {$data['email']} via data manager");
    }

    /**
     * Import services from CSV
     */
    public function importServices(string $filepath): array
    {
        $handle = fopen($filepath, 'r');
        $headers = fgetcsv($handle);

        $requiredFields = ['client_email', 'domain', 'product_name'];
        $this->validateHeaders($headers, $requiredFields);

        if (!empty($this->errors)) {
            return $this->getResult();
        }

        $lineCount = 1;
        while (($row = fgetcsv($handle)) !== false) {
            $lineCount++;
            $data = array_combine($headers, $row);

            try {
                $this->importService($data);
                $this->importedCount++;
            } catch (Exception $e) {
                $this->errors[] = "Line {$lineCount}: " . $e->getMessage();
            }
        }

        fclose($handle);
        return $this->getResult();
    }

    private function importService(array $data): void
    {
        // Find client
        $client = Capsule::table('tblclients')
            ->where('email', $data['client_email'])
            ->first();

        if (!$client) {
            throw new Exception("Client not found: {$data['client_email']}");
        }

        // Find product
        $product = Capsule::table('tblproducts')
            ->where('name', $data['product_name'])
            ->first();

        if (!$product) {
            throw new Exception("Product not found: {$data['product_name']}");
        }

        // Find server
        $serverId = null;
        if (!empty($data['server_name'])) {
            $server = Capsule::table('tblservers')
                ->where('name', $data['server_name'])
                ->first();
            $serverId = $server->id ?? null;
        }

        // Create service
        $serviceData = [
            'userid'       => $client->id,
            'packageid'    => $product->id,
            'server'       => $serverId,
            'domain'       => $data['domain'],
            'regdate'      => date('Y-m-d H:i:s'),
            'domainstatus' => 'Pending',
            'billingcycle' => $data['billing_cycle'] ?? 'Monthly',
            'amount'       => $data['amount'] ?? $product->monthly,
        ];

        $serviceId = Capsule::table('tblhosting')->insertGetId($serviceData);

        logActivity("Imported service {$data['domain']} for {$data['client_email']}");

        // Optionally create order
        if (isset($data['create_order']) && $data['create_order']) {
            $this->createServiceOrder($serviceId, $client, $data);
        }
    }

    private function createServiceOrder(int $serviceId, $client, array $data): void
    {
        $orderId = Capsule::table('tblorders')->insertGetId([
            'ordernum'     => createOrderNumber(),
            'userid'        => $client->id,
            'date'          => date('Y-m-d H:i:s'),
            'status'       => ($data['activate'] ?? false) ? 'Active' : 'Pending',
        ]);

        Capsule::table('tblorderitems')->insert([
            'orderid'    => $orderId,
            'type'       => 'hosting',
            'relid'      => $serviceId,
            'userid'     => $client->id,
        ]);
    }

    private function validateHeaders(array $headers, array $required): void
    {
        foreach ($required as $field) {
            if (!in_array($field, $headers)) {
                $this->errors[] = "Missing required column: {$field}";
            }
        }
    }

    private function getResult(): array
    {
        return [
            'success'      => empty($this->errors),
            'imported'     => $this->importedCount,
            'skipped'      => $this->skippedCount,
            'errors'       => $this->errors,
            'warnings'     => $this->warnings,
        ];
    }
}
```

### Step 3: Implement Database Migration Support

Handle complex data migrations:

```php
// includes/classes/DatabaseMigration.php

class DatabaseMigration
{
    /**
     * Migrate data between WHMCS installations
     */
    public function migrateToNewInstallation(array $options): array
    {
        $exporter = new DataExporter();
        $backupResult = $this->createFullBackup($exporter);

        logActivity("Starting data migration at " . date('Y-m-d H:i:s'));

        $migrationResults = [
            'clients'     => $exporter->exportClients($options),
            'services'    => $exporter->exportServices($options),
            'invoices'    => $exporter->exportInvoices($options),
            'transactions'=> $exporter->exportTransactions($options),
            'domains'     => $exporter->exportDomains(),
        ];

        $this->createMigrationManifest($migrationResults);

        return $migrationResults;
    }

    private function createFullBackup(DataExporter $exporter): array
    {
        $backupPath = ROOTDIR . '/storage/backups/migration_' . date('Ymd_His') . '/';
        mkdir($backupPath, 0755, true);

        // Database backup
        $dbBackup = $this->backupDatabase($backupPath . 'database.sql');

        // Full data export
        $dataBackup = $exporter->exportAll();

        return [
            'path'      => $backupPath,
            'database'  => $dbBackup,
            'data'      => $dataBackup,
        ];
    }

    private function backupDatabase(string $filepath): bool
    {
        $pdo = Capsule::connection()->getPdo();
        $tables = $pdo->query("SHOW TABLES")->fetchAll(PDO::FETCH_COLUMN);

        $output = "";
        foreach ($tables as $table) {
            $result = $pdo->query("SELECT * FROM {$table}");
            $fields = $result->fetchAll(PDO::FETCH_COLUMN);
            $output .= "DROP TABLE IF EXISTS {$table};\n";

            $create = $pdo->query("SHOW CREATE TABLE {$table}")->fetch();
            $output .= $create[1] . ";\n\n";

            while ($row = $result->fetch(PDO::FETCH_ASSOC)) {
                $values = array_map([$pdo, 'quote'], array_values($row));
                $output .= "INSERT INTO {$table} (" . implode(',', $fields) . ") VALUES (" . implode(',', $values) . ");\n";
            }
            $output .= "\n";
        }

        file_put_contents($filepath, $output);
        return true;
    }

    private function createMigrationManifest(array $results): void
    {
        $manifest = [
            'migration_date' => date('Y-m-d H:i:s'),
            'source_version' => Capsule::table('tblconfiguration')
                ->where('setting', 'Version')->first()->value ?? 'Unknown',
            'files' => $results,
        ];

        file_put_contents(
            ROOTDIR . '/storage/backups/manifest.json',
            json_encode($manifest, JSON_PRETTY_PRINT)
        );
    }
}
```

### Step 4: Set Up Automated Backup Schedule

Implement scheduled backups:

```php
// includes/hooks/automated_backup.php

add_hook('DailyCronJob', 1, function(array $vars) {
    $backupScheduler = new AutomatedBackupScheduler();
    return $backupScheduler->runDailyBackup();
});

class AutomatedBackupScheduler
{
    private string $backupPath;
    private int $retentionDays = 30;

    public function __construct()
    {
        $this->backupPath = ROOTDIR . '/storage/automated_backups/';
        if (!is_dir($this->backupPath)) {
            mkdir($this->backupPath, 0755, true);
        }
    }

    public function runDailyBackup(): void
    {
        $timestamp = date('Y-m-d_His');
        $backupDir = $this->backupPath . 'backup_' . $timestamp . '/';
        mkdir($backupDir, 0755, true);

        $this->backupDatabase($backupDir . 'whmcs_database.sql');
        $this->backupFiles($backupDir . 'whmcs_files.zip');
        $this->cleanupOldBackups();

        logActivity("Daily automated backup completed: {$timestamp}");
    }

    private function backupDatabase(string $filepath): void
    {
        $pdo = Capsule::connection()->getPdo();
        $output = `mysqldump -h {$_ENV['DB_HOST']} -u {$_ENV['DB_USERNAME']} -p{$_ENV['DB_PASSWORD']} {$_ENV['DB_NAME']} > {$filepath}`;
    }

    private function backupFiles(string $filepath): void
    {
        $zip = new ZipArchive();
        $zip->open($filepath, ZipArchive::CREATE);

        $directories = [
            ROOTDIR . '/templates/',
            ROOTDIR . '/attachments/',
            ROOTDIR . '/downloads/',
        ];

        foreach ($directories as $dir) {
            if (is_dir($dir)) {
                $this->addDirectoryToZip($zip, $dir, ROOTDIR);
            }
        }

        $zip->close();
    }

    private function addDirectoryToZip(ZipArchive $zip, string $dir, string $basePath): void
    {
        $files = new RecursiveIteratorIterator(
            new RecursiveDirectoryIterator($dir)
        );

        foreach ($files as $file) {
            if ($file->isDir()) continue;

            $filePath = $file->getRealPath();
            $localPath = str_replace($basePath . '/', '', $filePath);
            $zip->addFile($filePath, $localPath);
        }
    }

    private function cleanupOldBackups(): void
    {
        $cutoffDate = strtotime("-{$this->retentionDays} days");
        $backups = glob($this->backupPath . 'backup_*');

        foreach ($backups as $backup) {
            if (is_dir($backup)) {
                $backupTime = filemtime($backup);
                if ($backupTime < $cutoffDate) {
                    $this->removeDirectory($backup);
                }
            }
        }
    }

    private function removeDirectory(string $dir): void
    {
        $files = array_diff(scandir($dir), ['.', '..']);
        foreach ($files as $file) {
            $path = $dir . '/' . $file;
            is_dir($path) ? $this->removeDirectory($path) : unlink($path);
        }
        rmdir($dir);
    }
}
```

---

## Best Practices

1. **Always backup before import** - Create database backup before any import operation
2. **Test on staging first** - Validate import results in development environment
3. **Use transactions** - Wrap bulk operations in database transactions
4. **Validate data format** - Check CSV headers and data types before import
5. **Handle duplicates gracefully** - Decide how to handle existing records
6. **Log all operations** - Track every import operation for audit trail
7. **Clean up temporary files** - Remove import files after processing
8. **Verify data integrity** - Check record counts and checksums after import
9. **Schedule during off-peak** - Run large imports during low-traffic periods
10. **Maintain rollback capability** - Keep original backup until import verified

---

## Verification Checklist

- [ ] Database backup completed before import
- [ ] CSV files validated with correct headers
- [ ] Required field validation working
- [ ] Client import creates appropriate records
- [ ] Service import links to correct client and product
- [ ] Duplicate handling verified
- [ ] Transaction logging captures all changes
- [ ] Imported count matches expected records
- [ ] Data integrity checks pass
- [ ] Automated backup schedule runs successfully
