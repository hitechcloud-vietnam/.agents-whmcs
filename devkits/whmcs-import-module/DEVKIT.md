# WHMCS Import Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-import-module/
├── import.php             # Import controller
├── lib/
│   ├── ImportHandler.php  # Import logic
│   ├── CsvParser.php      # CSV parsing
│   └── DataMapper.php      # Data mapping
├── templates/
│   ├── admin.tpl          # Admin interface
│   └── mapping.tpl        # Field mapping
└── import-mappings.php    # Predefined mappings
```

## Import Controller Template

```php
<?php
/**
 * WHMCS Import Module: {Module}
 * DevKit Template
 * 
 * Installation: Upload to modules/addons/{module}/
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

use WHMCS\Database\Capsule;

/**
 * Config function (required for addon modules)
 */
function {module}_config(): array {
    return [
        'name' => '{Import Module}',
        'description' => 'Import data from external sources',
        'version' => '1.0',
        'author' => '{Author}',
    ];
}

/**
 * Activate (required for addon modules)
 */
function {module}_activate(): array {
    // Create import log table
    Capsule::schema()->create('mod_{module}_import_log', function($t) {
        $t->increments('id');
        $t->string('import_type');
        $t->string('filename');
        $t->integer('total_rows');
        $t->integer('imported_rows');
        $t->integer('failed_rows');
        $t->text('error_log');
        $t->timestamp('imported_at');
    });
    
    // Create import settings table
    Capsule::schema()->create('mod_{module}_import_settings', function($t) {
        $t->increments('id');
        $t->string('setting_name');
        $t->text('setting_value');
        $t->timestamp('updated_at');
    });
    
    return ['status' => 'success', 'description' => 'Module activated'];
}

/**
 * Deactivate
 */
function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_{module}_import_log');
    Capsule::schema()->dropIfExists('mod_{module}_import_settings');
    
    return ['status' => 'success', 'description' => 'Module deactivated'];
}

/**
 * Output function
 */
function {module}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'list';
    
    switch ($action) {
        case 'upload':
            {module}_showUploadForm();
            break;
        case 'mapping':
            {module}_showMappingForm();
            break;
        case 'preview':
            {module}_showPreviewTable();
            break;
        case 'import':
            {module}_executeImport();
            break;
        case 'history':
            {module}_showHistory();
            break;
        default:
            {module}_showDashboard();
    }
}

/**
 * Show Dashboard
 */
function {module}_showDashboard(): void {
    echo <<<'HTML'
<div class="import-module">
    <div class="row">
        <div class="col-md-12">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Data Import</h3>
                </div>
                <div class="panel-body">
                    <p>Import clients, services, or other data from CSV files.</p>
                    
                    <div class="btn-group">
                        <a href="?module={module}&action=upload&type=clients" 
                           class="btn btn-primary">
                            <i class="fa fa-users"></i> Import Clients
                        </a>
                        <a href="?module={module}&action=upload&type=services" 
                           class="btn btn-primary">
                            <i class="fa fa-server"></i> Import Services
                        </a>
                        <a href="?module={module}&action=upload&type=invoices" 
                           class="btn btn-primary">
                            <i class="fa fa-file-invoice"></i> Import Invoices
                        </a>
                        <a href="?module={module}&action=history" 
                           class="btn btn-default">
                            <i class="fa fa-history"></i> Import History
                        </a>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>
HTML;
}

/**
 * Show Upload Form
 */
function {module}_showUploadForm(): void {
    $type = $_GET['type'] ?? 'clients';
    
    echo <<<HTML
<div class="upload-form">
    <div class="row">
        <div class="col-md-8">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Upload {$type} Data</h3>
                </div>
                <div class="panel-body">
                    <form method="post" enctype="multipart/form-data" 
                          action="?module={module}&action=mapping">
                        <input type="hidden" name="type" value="{$type}">
                        <input type="hidden" name="csrf_token" value="{$_SESSION['csrf_token']}">
                        
                        <div class="form-group">
                            <label for="csv_file">CSV File</label>
                            <input type="file" name="csv_file" id="csv_file" 
                                   class="form-control" accept=".csv" required>
                            <p class="help-block">
                                Maximum file size: 10MB. Required columns depend on import type.
                            </p>
                        </div>
                        
                        <div class="form-group">
                            <label>
                                <input type="checkbox" name="has_header" value="1" checked>
                                First row contains headers
                            </label>
                        </div>
                        
                        <div class="form-group">
                            <label for="delimiter">Delimiter</label>
                            <select name="delimiter" id="delimiter" class="form-control">
                                <option value=",">Comma (,)</option>
                                <option value=";">Semicolon (;)</option>
                                <option value="\t">Tab</option>
                            </select>
                        </div>
                        
                        <div class="form-group">
                            <label for="enclosure">Enclosure</label>
                            <select name="enclosure" id="enclosure" class="form-control">
                                <option value='"'>Double Quote (")</option>
                                <option value="'">Single Quote (')</option>
                            </select>
                        </div>
                        
                        <div class="form-group">
                            <a href="?module={module}" class="btn btn-default">Cancel</a>
                            <button type="submit" class="btn btn-primary">
                                <i class="fa fa-upload"></i> Upload & Continue
                            </button>
                        </div>
                    </form>
                </div>
            </div>
        </div>
        
        <div class="col-md-4">
            <div class="panel panel-info">
                <div class="panel-heading">
                    <h3 class="panel-title">Required Columns</h3>
                </div>
                <div class="panel-body">
                    <h4>Clients</h4>
                    <ul>
                        <li>firstname</li>
                        <li>lastname</li>
                        <li>email</li>
                    </ul>
                    
                    <h4>Services</h4>
                    <ul>
                        <li>client_email</li>
                        <li>product_name</li>
                        <li>domain</li>
                    </ul>
                    
                    <h4>Invoices</h4>
                    <ul>
                        <li>client_email</li>
                        <li>amount</li>
                        <li>duedate</li>
                    </ul>
                </div>
            </div>
        </div>
    </div>
</div>
HTML;
}
```

## Import Handler Class

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class ImportHandler {
    
    private string $importType;
    private array $mappings;
    private array $data;
    private int $imported = 0;
    private int $failed = 0;
    private array $errors = [];
    
    public function __construct(string $importType, array $mappings = []) {
        $this->importType = $importType;
        $this->mappings = $mappings;
    }
    
    public function setData(array $data): void {
        $this->data = $data;
    }
    
    public function execute(): array {
        $method = 'import' . ucfirst($this->importType);
        
        if (!method_exists($this, $method)) {
            throw new \Exception("Import method not found: {$method}");
        }
        
        foreach ($this->data as $index => $row) {
            try {
                $this->$method($row);
                $this->imported++;
            } catch (\Exception $e) {
                $this->failed++;
                $this->errors[] = [
                    'row' => $index + 1,
                    'error' => $e->getMessage(),
                    'data' => $row,
                ];
            }
        }
        
        $this->logImport();
        
        return [
            'total' => count($this->data),
            'imported' => $this->imported,
            'failed' => $this->failed,
            'errors' => $this->errors,
        ];
    }
    
    private function importClients(array $row): void {
        // Map fields
        $data = $this->mapFields($row, [
            'firstname' => 'first_name',
            'lastname' => 'last_name',
            'email' => 'email',
            'companyname' => 'company_name',
            'address1' => 'address1',
            'city' => 'city',
            'state' => 'state',
            'postcode' => 'postcode',
            'country' => 'country',
            'phonenumber' => 'phone',
        ]);
        
        // Check if client exists
        $existing = Capsule::table('tblclients')
            ->where('email', $data['email'])
            ->first();
        
        if ($existing) {
            throw new \Exception("Client already exists: {$data['email']}");
        }
        
        // Create client
        $password = !empty($data['password']) 
            ? $data['password'] 
            : generate_random_string();
        
        $clientId = Capsule::table('tblclients')->insertGetId([
            'firstname' => $data['first_name'] ?? '',
            'lastname' => $data['last_name'] ?? '',
            'email' => $data['email'],
            'companyname' => $data['company_name'] ?? '',
            'address1' => $data['address1'] ?? '',
            'city' => $data['city'] ?? '',
            'state' => $data['state'] ?? '',
            'postcode' => $data['postcode'] ?? '',
            'country' => $data['country'] ?? 'US',
            'phonenumber' => $data['phone'] ?? '',
            'password' => encrypt($password),
            'created_at' => date('Y-m-d H:i:s'),
            'notes' => 'Imported via ' . $this->importType . ' module',
        ]);
        
        logActivity("{Module}: Imported client ID {$clientId} - {$data['email']}");
    }
    
    private function importServices(array $row): void {
        $data = $this->mapFields($row, [
            'client_email' => 'client_email',
            'product_name' => 'product_name',
            'domain' => 'domain',
            'billing_cycle' => 'billing_cycle',
            'regdate' => 'regdate',
        ]);
        
        // Find client
        $client = Capsule::table('tblclients')
            ->where('email', $data['client_email'])
            ->first();
        
        if (!$client) {
            throw new \Exception("Client not found: {$data['client_email']}");
        }
        
        // Find product
        $product = Capsule::table('tblproducts')
            ->where('name', $data['product_name'])
            ->first();
        
        if (!$product) {
            throw new \Exception("Product not found: {$data['product_name']}");
        }
        
        $billingCycle = $data['billing_cycle'] ?? 'Monthly';
        
        Capsule::table('tblhosting')->insert([
            'userid' => $client->id,
            'packageid' => $product->id,
            'domain' => $data['domain'] ?? '',
            'regdate' => $data['regdate'] ?? date('Y-m-d'),
            'domainstatus' => 'Active',
            'billingcycle' => $billingCycle,
            'nextduedate' => date('Y-m-d', strtotime('+1 month')),
        ]);
    }
    
    private function mapFields(array $row, array $mapping): array {
        $mapped = [];
        
        foreach ($mapping as $source => $target) {
            $mapped[$target] = $row[$source] ?? null;
        }
        
        return $mapped;
    }
    
    private function logImport(): void {
        Capsule::table('mod_{module}_import_log')->insert([
            'import_type' => $this->importType,
            'filename' => $_SESSION['import_filename'] ?? 'N/A',
            'total_rows' => count($this->data),
            'imported_rows' => $this->imported,
            'failed_rows' => $this->failed,
            'error_log' => json_encode($this->errors),
            'imported_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    public function getImportedCount(): int {
        return $this->imported;
    }
    
    public function getFailedCount(): int {
        return $this->failed;
    }
    
    public function getErrors(): array {
        return $this->errors;
    }
}
```

## CSV Parser Class

```php
<?php
namespace {Module};

class CsvParser {
    
    private string $delimiter = ',';
    private string $enclosure = '"';
    private bool $hasHeader = true;
    
    public function __construct(string $delimiter = ',', string $enclosure = '"') {
        $this->delimiter = $delimiter;
        $this->enclosure = $enclosure;
    }
    
    public function setHasHeader(bool $hasHeader): void {
        $this->hasHeader = $hasHeader;
    }
    
    public function parse(string $filePath): array {
        $data = [];
        $headers = [];
        
        if (($handle = fopen($filePath, 'r')) === false) {
            throw new \Exception("Cannot open file: {$filePath}");
        }
        
        $row = 0;
        while (($fields = fgetcsv($handle, 0, $this->delimiter, $this->enclosure)) !== false) {
            $row++;
            
            if ($this->hasHeader && $row === 1) {
                $headers = array_map('trim', $fields);
                continue;
            }
            
            if (empty(array_filter($fields))) {
                continue; // Skip empty rows
            }
            
            if ($this->hasHeader) {
                $data[] = array_combine($headers, array_map('trim', $fields));
            } else {
                $data[] = array_map('trim', $fields);
            }
        }
        
        fclose($handle);
        
        return $data;
    }
    
    public function parseString(string $content): array {
        $tempFile = tempnam(sys_get_temp_dir(), 'csv_');
        file_put_contents($tempFile, $content);
        
        try {
            return $this->parse($tempFile);
        } finally {
            unlink($tempFile);
        }
    }
    
    public function getHeaders(string $filePath): array {
        $handle = fopen($filePath, 'r');
        $headers = fgetcsv($handle, 0, $this->delimiter, $this->enclosure);
        fclose($handle);
        
        return array_map('trim', $headers);
    }
    
    public function validate(array $data, array $requiredFields): array {
        $errors = [];
        
        foreach ($requiredFields as $field) {
            $found = false;
            foreach ($data as $row) {
                if (isset($row[$field]) && !empty($row[$field])) {
                    $found = true;
                    break;
                }
            }
            
            if (!$found) {
                $errors[] = "Required field missing: {$field}";
            }
        }
        
        return $errors;
    }
}
```

## Checklist

```
Pre-Dev:
□ Define import types (clients, services, invoices, etc.)
□ Identify required columns for each type
□ Plan field mapping options
□ Design validation rules
□ Plan error handling strategy

Development:
□ Create module activate/deactivate functions
□ Create import tables (log, settings)
□ Create CSV Parser class
□ Create Import Handler class
□ Implement importClients method
□ Implement importServices method
□ Implement importInvoices method
□ Add field mapping functionality
□ Add preview functionality
□ Create admin templates
□ Add progress tracking
□ Add error logging

Testing:
□ Test CSV file parsing
□ Test with various delimiters
□ Test field mapping
□ Test client import
□ Test service import
□ Test duplicate detection
□ Test error handling
□ Test rollback on failure
□ Test with large files
□ Verify data integrity
```