# WHMCS Data Export Utilities Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Comprehensive data export utilities with scheduling, formats, and filtering.

## Database Schema

```php
<?php
// modules/addons/data_export/data_export.php

use WHMCS\Database\Capsule;

function data_export_config(): array {
    return [
        'name' => 'Data Export',
        'description' => 'Export WHMCS data in various formats',
        'version' => '1.0',
    ];
}

function data_export_activate(): array {
    Capsule::schema()->create('mod_export_jobs', function($t) {
        $t->increments('id');
        $t->string('job_name', 100);
        $t->string('export_type', 50);
        $t->text('filters')->nullable();
        $t->string('format', 20)->default('csv');
        $t->string('status', 20)->default('pending');
        $t->integer('created_by')->unsigned();
        $t->integer('file_size')->unsigned()->default(0);
        $t->string('file_path', 255)->nullable();
        $t->string('download_token', 64)->nullable();
        $t->timestamp('completed_at')->nullable();
        $t->timestamps();
    });

    Capsule::schema()->create('mod_export_schedules', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('export_type', 50);
        $t->text('filters')->nullable();
        $t->string('format', 20)->default('csv');
        $t->string('schedule', 50);
        $t->string('recipients', 500)->nullable();
        $t->boolean('is_active')->default(true);
        $t->timestamp('last_run')->nullable();
        $t->timestamp('next_run')->nullable();
        $t->timestamps();
    });

    Capsule::schema()->create('mod_export_templates', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('export_type', 50);
        $t->text('column_config')->nullable();
        $t->text('filters')->nullable();
        $t->string('format', 20)->default('csv');
        $t->boolean('is_system')->default(false);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_export_logs', function($t) {
        $t->increments('id');
        $t->integer('job_id')->unsigned();
        $t->string('action', 100);
        $t->text('details')->nullable();
        $t->integer('records_processed')->default(0);
        $t->timestamp('executed_at')->useCurrent();
    });

    // Create default templates
    $templates = [
        ['name' => 'Clients Full Export', 'export_type' => 'clients', 'format' => 'csv'],
        ['name' => 'Invoices Export', 'export_type' => 'invoices', 'format' => 'csv'],
        ['name' => 'Services Export', 'export_type' => 'services', 'format' => 'csv'],
        ['name' => 'Orders Export', 'export_type' => 'orders', 'format' => 'csv'],
    ];

    foreach ($templates as $template) {
        Capsule::table('mod_export_templates')->insert($template);
    }

    return ['status' => 'success'];
}

function data_export_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_export_logs');
    Capsule::schema()->dropIfExists('mod_export_templates');
    Capsule::schema()->dropIfExists('mod_export_schedules');
    Capsule::schema()->dropIfExists('mod_export_jobs');
    return ['status' => 'success'];
}
```

## Data Export Manager

```php
<?php
class DataExportManager {
    private array $exportHandlers = [];

    public function __construct() {
        $this->exportHandlers = [
            'clients' => [$this, 'exportClients'],
            'invoices' => [$this, 'exportInvoices'],
            'services' => [$this, 'exportServices'],
            'orders' => [$this, 'exportOrders'],
            'transactions' => [$this, 'exportTransactions'],
            'tickets' => [$this, 'exportTickets'],
            'domains' => [$this, 'exportDomains'],
            'products' => [$this, 'exportProducts'],
        ];
    }

    public function createExportJob(string $exportType, array $filters, string $format = 'csv'): int {
        $jobId = Capsule::table('mod_export_jobs')->insertGetId([
            'job_name' => ucfirst($exportType) . ' Export',
            'export_type' => $exportType,
            'filters' => json_encode($filters),
            'format' => $format,
            'created_by' => $_SESSION['adminid'] ?? 0,
            'status' => 'pending',
        ]);

        return $jobId;
    }

    public function processExport(int $jobId): array {
        $job = Capsule::table('mod_export_jobs')->find($jobId);

        if (!$job) {
            return ['success' => false, 'error' => 'Job not found'];
        }

        Capsule::table('mod_export_jobs')
            ->where('id', $jobId)
            ->update(['status' => 'processing']);

        $this->logAction($jobId, 'started', "Processing {$job->export_type} export");

        try {
            $filters = json_decode($job->filters, true) ?? [];
            $handler = $this->exportHandlers[$job->export_type] ?? null;

            if (!$handler) {
                throw new \Exception("Export type not supported: {$job->export_type}");
            }

            $data = $handler($filters);

            $filename = $this->generateFilename($job->export_type, $job->format);
            $filePath = $this->getExportDir() . '/' . $filename;

            $result = $this->writeFile($filePath, $data, $job->format);

            $downloadToken = bin2hex(random_bytes(32));

            Capsule::table('mod_export_jobs')
                ->where('id', $jobId)
                ->update([
                    'status' => 'completed',
                    'file_path' => $filePath,
                    'file_size' => filesize($filePath),
                    'download_token' => $downloadToken,
                    'completed_at' => date('Y-m-d H:i:s'),
                ]);

            $this->logAction($jobId, 'completed', "Export completed", count($data));

            return [
                'success' => true,
                'file_path' => $filePath,
                'download_token' => $downloadToken,
                'record_count' => count($data),
            ];

        } catch (\Exception $e) {
            Capsule::table('mod_export_jobs')
                ->where('id', $jobId)
                ->update(['status' => 'failed']);

            $this->logAction($jobId, 'failed', $e->getMessage());

            return [
                'success' => false,
                'error' => $e->getMessage(),
            ];
        }
    }

    private function exportClients(array $filters): array {
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

        if (!empty($filters['search'])) {
            $search = $filters['search'];
            $query->where(function($q) use ($search) {
                $q->where('firstname', 'like', "%{$search}%")
                  ->orWhere('lastname', 'like', "%{$search}%")
                  ->orWhere('email', 'like', "%{$search}%");
            });
        }

        $clients = $query->get();

        $data = [];
        foreach ($clients as $client) {
            $data[] = [
                'id' => $client->id,
                'firstname' => $client->firstname,
                'lastname' => $client->lastname,
                'email' => $client->email,
                'company' => $client->companyname,
                'phone' => $client->phonenumber,
                'created' => $client->datecreated,
                'status' => $client->status,
            ];
        }

        return $data;
    }

    private function exportInvoices(array $filters): array {
        $query = Capsule::table('tblinvoices');

        if (!empty($filters['status'])) {
            $query->where('status', $filters['status']);
        }

        if (!empty($filters['date_from'])) {
            $query->where('date', '>=', $filters['date_from']);
        }

        if (!empty($filters['date_to'])) {
            $query->where('date', '<=', $filters['date_to']);
        }

        $invoices = $query->get();

        $data = [];
        foreach ($invoices as $invoice) {
            $user = Capsule::table('tblclients')->find($invoice->userid);

            $data[] = [
                'id' => $invoice->id,
                'invoice_number' => $invoice->invoicenum,
                'user_id' => $invoice->userid,
                'client_name' => $user ? $user->firstname . ' ' . $user->lastname : '',
                'client_email' => $user ? $user->email : '',
                'date' => $invoice->date,
                'due_date' => $invoice->duedate,
                'total' => $invoice->total,
                'status' => $invoice->status,
            ];
        }

        return $data;
    }

    private function exportServices(array $filters): array {
        $query = Capsule::table('tblhosting');

        if (!empty($filters['status'])) {
            $query->where('domainstatus', $filters['status']);
        }

        if (!empty($filters['product_id'])) {
            $query->where('packageid', $filters['product_id']);
        }

        $services = $query->get();

        $data = [];
        foreach ($services as $service) {
            $user = Capsule::table('tblclients')->find($service->userid);
            $product = Capsule::table('tblproducts')->find($service->packageid);

            $data[] = [
                'id' => $service->id,
                'user_id' => $service->userid,
                'client_name' => $user ? $user->firstname . ' ' . $user->lastname : '',
                'client_email' => $user ? $user->email : '',
                'product' => $product ? $product->name : '',
                'domain' => $service->domain,
                'reg_date' => $service->regdate,
                'next_due' => $service->nextduedate,
                'status' => $service->domainstatus,
            ];
        }

        return $data;
    }

    private function exportOrders(array $filters): array {
        $query = Capsule::table('tblorders');

        if (!empty($filters['status'])) {
            $query->where('status', $filters['status']);
        }

        if (!empty($filters['date_from'])) {
            $query->where('date', '>=', $filters['date_from']);
        }

        $orders = $query->get();

        $data = [];
        foreach ($orders as $order) {
            $user = Capsule::table('tblclients')->find($order->userid);

            $data[] = [
                'id' => $order->id,
                'order_number' => $order->ordernum,
                'user_id' => $order->userid,
                'client_name' => $user ? $user->firstname . ' ' . $user->lastname : '',
                'date' => $order->date,
                'amount' => $order->amount,
                'status' => $order->status,
            ];
        }

        return $data;
    }

    private function exportTransactions(array $filters): array {
        $query = Capsule::table('tblaccounts');

        if (!empty($filters['date_from'])) {
            $query->where('date', '>=', $filters['date_from']);
        }

        if (!empty($filters['date_to'])) {
            $query->where('date', '<=', $filters['date_to']);
        }

        $transactions = $query->get();

        $data = [];
        foreach ($transactions as $trans) {
            $data[] = [
                'id' => $trans->id,
                'user_id' => $trans->userid,
                'date' => $trans->date,
                'description' => $trans->description,
                'amount' => $trans->amount,
                'fees' => $trans->fees,
                'type' => $trans->type,
            ];
        }

        return $data;
    }

    private function exportTickets(array $filters): array {
        $query = Capsule::table('tbltickets');

        if (!empty($filters['status'])) {
            $query->where('status', $filters['status']);
        }

        if (!empty($filters['priority'])) {
            $query->where('urgency', $filters['priority']);
        }

        $tickets = $query->get();

        $data = [];
        foreach ($tickets as $ticket) {
            $user = Capsule::table('tblclients')->find($ticket->userid);

            $data[] = [
                'id' => $ticket->id,
                'ticket_number' => $ticket->tid,
                'user_id' => $ticket->userid,
                'client_name' => $user ? $user->firstname . ' ' . $user->lastname : '',
                'subject' => $ticket->title,
                'priority' => $ticket->urgency,
                'status' => $ticket->status,
                'created' => $ticket->created,
            ];
        }

        return $data;
    }

    private function exportDomains(array $filters): array {
        $query = Capsule::table('tbldomains');

        if (!empty($filters['status'])) {
            $query->where('status', $filters['status']);
        }

        $domains = $query->get();

        $data = [];
        foreach ($domains as $domain) {
            $user = Capsule::table('tblclients')->find($domain->userid);

            $data[] = [
                'id' => $domain->id,
                'domain' => $domain->domain,
                'user_id' => $domain->userid,
                'client_name' => $user ? $user->firstname . ' ' . $user->lastname : '',
                'registration_date' => $domain->registrationdate,
                'expiry_date' => $domain->expirydate,
                'registrar' => $domain->registrar,
                'status' => $domain->status,
            ];
        }

        return $data;
    }

    private function exportProducts(array $filters): array {
        $products = Capsule::table('tblproducts')->get();

        $data = [];
        foreach ($products as $product) {
            $group = Capsule::table('tblproductgroups')->find($product->gid);

            $data[] = [
                'id' => $product->id,
                'name' => $product->name,
                'group' => $group ? $group->name : '',
                'type' => $product->type,
                'stock' => $product->stock,
                'description' => $product->description,
            ];
        }

        return $data;
    }

    private function writeFile(string $filePath, array $data, string $format): bool {
        $this->ensureExportDir();

        switch ($format) {
            case 'csv':
                return $this->writeCsv($filePath, $data);
            case 'json':
                return $this->writeJson($filePath, $data);
            case 'xml':
                return $this->writeXml($filePath, $data);
            case 'xlsx':
                return $this->writeExcel($filePath, $data);
            default:
                return $this->writeCsv($filePath, $data);
        }
    }

    private function writeCsv(string $filePath, array $data): bool {
        if (empty($data)) {
            file_put_contents($filePath, '');
            return true;
        }

        $handle = fopen($filePath, 'w');

        // Header row
        fputcsv($handle, array_keys($data[0]));

        // Data rows
        foreach ($data as $row) {
            fputcsv($handle, $row);
        }

        fclose($handle);
        return true;
    }

    private function writeJson(string $filePath, array $data): bool {
        $json = json_encode($data, JSON_PRETTY_PRINT);
        file_put_contents($filePath, $json);
        return true;
    }

    private function writeXml(string $filePath, array $data): bool {
        $xml = new \SimpleXMLElement('<export/>');

        foreach ($data as $item) {
            $itemElement = $xml->addChild('item');
            foreach ($item as $key => $value) {
                $itemElement->addChild($key, htmlspecialchars($value ?? ''));
            }
        }

        $xml->asXML($filePath);
        return true;
    }

    private function writeExcel(string $filePath, array $data): bool {
        // For Excel, we'll create a CSV that Excel can open
        // Full Excel support would require PhpSpreadsheet library
        return $this->writeCsv($filePath, $data);
    }

    private function generateFilename(string $type, string $format): string {
        return "{$type}_export_" . date('Y-m-d_His') . ".{$format}";
    }

    private function getExportDir(): string {
        $dir = ini_get('upload_tmp_dir') ?: sys_get_temp_dir();
        $dir .= '/whmcs_exports';

        if (!is_dir($dir)) {
            mkdir($dir, 0755, true);
        }

        return $dir;
    }

    private function ensureExportDir(): void {
        $dir = $this->getExportDir();
        if (!is_dir($dir)) {
            mkdir($dir, 0755, true);
        }
    }

    private function logAction(int $jobId, string $action, string $details, int $records = 0): void {
        Capsule::table('mod_export_logs')->insert([
            'job_id' => $jobId,
            'action' => $action,
            'details' => $details,
            'records_processed' => $records,
        ]);
    }

    public function getJob(int $jobId): ?object {
        return Capsule::table('mod_export_jobs')->find($jobId);
    }

    public function getPendingJobs(): array {
        return Capsule::table('mod_export_jobs')
            ->where('status', 'pending')
            ->orderBy('created_at')
            ->get();
    }

    public function downloadFile(string $token): ?string {
        $job = Capsule::table('mod_export_jobs')
            ->where('download_token', $token)
            ->first();

        if (!$job || !$job->file_path || !file_exists($job->file_path)) {
            return null;
        }

        return $job->file_path;
    }

    public function cleanupOldFiles(int $days = 7): int {
        $cutoff = time() - ($days * 86400);
        $count = 0;

        $jobs = Capsule::table('mod_export_jobs')
            ->where('status', 'completed')
            ->where('completed_at', '<', date('Y-m-d H:i:s', $cutoff))
            ->get();

        foreach ($jobs as $job) {
            if ($job->file_path && file_exists($job->file_path)) {
                unlink($job->file_path);
                $count++;
            }

            Capsule::table('mod_export_jobs')
                ->where('id', $job->id)
                ->update(['file_path' => null]);
        }

        return $count;
    }

    public function scheduleExport(string $name, string $exportType, string $schedule, array $filters, string $format = 'csv', ?string $recipients = null): int {
        return Capsule::table('mod_export_schedules')->insertGetId([
            'name' => $name,
            'export_type' => $exportType,
            'filters' => json_encode($filters),
            'format' => $format,
            'schedule' => $schedule,
            'recipients' => $recipients,
            'next_run' => $this->calculateNextRun($schedule),
        ]);
    }

    private function calculateNextRun(string $schedule): string {
        // Simple cron parsing
        $parts = explode(' ', $schedule);
        $minute = $parts[0] ?? '0';
        $hour = $parts[1] ?? '0';
        $day = $parts[2] ?? '*';
        $month = $parts[3] ?? '*';
        $weekday = $parts[4] ?? '*';

        // For daily at specific time
        if ($day === '*' && $month === '*' && $weekday === '*') {
            return date('Y-m-d') . ' ' . str_pad($hour, 2, '0', STR_PAD_LEFT) . ':' . str_pad($minute, 2, '0', STR_PAD_LEFT) . ':00';
        }

        return date('Y-m-d H:i:s', strtotime('+1 day'));
    }
}
```

## Cron Processing for Scheduled Exports

```php
<?php
add_hook('DailyCronJob', 1, function($vars) {
    $exportManager = new DataExportManager();

    // Process scheduled exports
    $schedules = Capsule::table('mod_export_schedules')
        ->where('is_active', 1)
        ->where('next_run', '<=', date('Y-m-d H:i:s'))
        ->get();

    foreach ($schedules as $schedule) {
        $filters = json_decode($schedule->filters, true) ?? [];

        // Create and process job
        $jobId = $exportManager->createExportJob($schedule->export_type, $filters, $schedule->format);
        $result = $exportManager->processExport($jobId);

        if ($result['success']) {
            // Send to recipients if configured
            if ($schedule->recipients) {
                $recipients = explode(',', $schedule->recipients);
                $job = $exportManager->getJob($jobId);

                foreach ($recipients as $email) {
                    sendTplEmail(trim($email), 'export_completed', [
                        'export_name' => $schedule->name,
                        'record_count' => $result['record_count'],
                        'download_link' => 'download.php?token=' . $result['download_token'],
                    ]);
                }
            }
        }

        // Calculate next run
        $nextRun = $exportManager->calculateNextRun($schedule->schedule);
        Capsule::table('mod_export_schedules')
            ->where('id', $schedule->id)
            ->update([
                'last_run' => date('Y-m-d H:i:s'),
                'next_run' => $nextRun,
            ]);
    }

    // Cleanup old files
    $exportManager->cleanupOldFiles();
});
```

## Admin Interface

```php
<?php
function data_export_output(array $vars): void {
    $action = $_GET['action'] ?? 'list';

    if ($action === 'export') {
        $exportType = $_GET['type'] ?? 'clients';

        if ($_SERVER['REQUEST_METHOD'] === 'POST') {
            check_token('WHMCS.admin.default');

            $filters = [
                'status' => $_POST['status'] ?? null,
                'date_from' => $_POST['date_from'] ?? null,
                'date_to' => $_POST['date_to'] ?? null,
                'search' => $_POST['search'] ?? null,
            ];

            $format = $_POST['format'] ?? 'csv';

            $manager = new DataExportManager();
            $jobId = $manager->createExportJob($exportType, $filters, $format);

            // Process synchronously for admin
            $result = $manager->processExport($jobId);

            if ($result['success']) {
                echo '<div class="alert alert-success">Export completed. <a href="?action=download&token=' . $result['download_token'] . '">Download</a></div>';
            } else {
                echo '<div class="alert alert-danger">Export failed: ' . $result['error'] . '</div>';
            }
        }

        echo '<h2>Export ' . ucfirst($exportType) . '</h2>';
        echo '<form method="post">';
        echo '<input type="hidden" name="_token" value="' . generate_token() . '">';

        echo '<div class="form-group"><label>Status</label>';
        echo '<select name="status" class="form-control"><option value="">All</option>';
        echo '<option value="Active">Active</option><option value="Inactive">Inactive</option>';
        echo '</select></div>';

        echo '<div class="form-group"><label>Date From</label>';
        echo '<input type="date" name="date_from" class="form-control"></div>';

        echo '<div class="form-group"><label>Date To</label>';
        echo '<input type="date" name="date_to" class="form-control"></div>';

        echo '<div class="form-group"><label>Format</label>';
        echo '<select name="format" class="form-control">';
        echo '<option value="csv">CSV</option>';
        echo '<option value="json">JSON</option>';
        echo '<option value="xml">XML</option>';
        echo '</select></div>';

        echo '<button type="submit" class="btn btn-primary">Export</button>';
        echo '</form>';
    } else {
        $jobs = Capsule::table('mod_export_jobs')
            ->orderBy('created_at', 'desc')
            ->limit(50)
            ->get();

        echo '<h2>Data Exports</h2>';
        echo '<a href="?action=export&type=clients" class="btn">Clients</a> ';
        echo '<a href="?action=export&type=invoices" class="btn">Invoices</a> ';
        echo '<a href="?action=export&type=services" class="btn">Services</a> ';
        echo '<a href="?action=export&type=orders" class="btn">Orders</a>';

        echo '<table class="datatable"><thead><tr>';
        echo '<th>Name</th><th>Type</th><th>Format</th><th>Status</th><th>Created</th><th>Actions</th>';
        echo '</tr></thead><tbody>';

        foreach ($jobs as $job) {
            echo '<tr>';
            echo "<td>{$job->job_name}</td>";
            echo "<td>{$job->export_type}</td>";
            echo "<td>{$job->format}</td>";
            echo "<td>{$job->status}</td>";
            echo "<td>{$job->created_at}</td>";
            echo '<td>' . ($job->download_token ? '<a href="?action=download&token=' . $job->download_token . '">Download</a>' : '-') . '</td>';
            echo '</tr>';
        }

        echo '</tbody></table>';
    }
}
```

---

**Related Skills:**
- whmcs-reporting
- whmcs-metrics-analytics
- whmcs-billing-dashboard