# WHMCS Data Import Workflow

## Purpose
Guide through importing data into WHMCS from various sources.

## Prerequisites
- WHMCS installation
- Admin access
- Data source prepared
- Backup completed

## Step-by-Step Process

### Step 1: Prepare Data Source

**Data Format Requirements:**
- CSV: UTF-8 encoded, comma or semicolon delimited
- XML: Well-formed with UTF-8 encoding
- JSON: Valid JSON with UTF-8 encoding

**Required Fields for Each Entity:**

**Clients:**
```csv
firstname,lastname,email,companyname,address1,city,state,postcode,country,phonenumber
John,Smith,john@example.com,Acme Inc,123 Main St,New York,NY,10001,US,555-1234
```

**Products/Services:**
```csv
client_email,product_name,billing_cycle,domain,next_due_date,custom_fields
john@example.com,Starter Hosting,monthly,example.com,2026-06-15,"""field1"":""value1"""
```

### Step 2: Create Import Script

**Create hooks/data_import.php:**
```php
<?php
/**
 * WHMCS Data Import System
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class DataImport {
    
    private $errors = [];
    private $imported = 0;
    private $skipped = 0;
    
    /**
     * Import clients from CSV
     */
    public function importClients($filePath, $options = []) {
        $defaults = [
            'delimiter' => ',',
            'enclosure' => '"',
            'has_header' => true,
            'update_existing' => false,
            'send_welcome' => false
        ];
        $options = array_merge($defaults, $options);
        
        $handle = fopen($filePath, 'r');
        if (!$handle) {
            throw new Exception("Cannot open file: $filePath");
        }
        
        $headers = [];
        $rowNum = 0;
        
        while (($data = fgetcsv($handle, 0, $options['delimiter'], $options['enclosure'])) !== false) {
            $rowNum++;
            
            // Skip header row
            if ($rowNum === 1 && $options['has_header']) {
                $headers = array_map('trim', $data);
                continue;
            }
            
            // Skip empty rows
            if (empty(array_filter($data))) {
                continue;
            }
            
            // Map data to headers
            $row = array_combine($headers, $data);
            
            try {
                $this->importClient($row, $options);
                $this->imported++;
            } catch (Exception $e) {
                $this->errors[] = "Row $rowNum: " . $e->getMessage();
                $this->skipped++;
            }
        }
        
        fclose($handle);
        
        return [
            'imported' => $this->imported,
            'skipped' => $this->skipped,
            'errors' => $this->errors
        ];
    }
    
    /**
     * Import single client
     */
    private function importClient($data, $options) {
        // Validate required fields
        $required = ['firstname', 'lastname', 'email'];
        foreach ($required as $field) {
            if (empty($data[$field])) {
                throw new Exception("Missing required field: $field");
            }
        }
        
        // Check if client exists
        $existing = Capsule::table('tblclients')
            ->where('email', $data['email'])
            ->first();
        
        if ($existing) {
            if (!$options['update_existing']) {
                throw new Exception("Client already exists: " . $data['email']);
            }
            
            // Update existing client
            $this->updateClient($existing->id, $data);
            return $existing->id;
        }
        
        // Create new client
        $clientData = [
            'firstname' => trim($data['firstname']),
            'lastname' => trim($data['lastname']),
            'email' => strtolower(trim($data['email'])),
            'companyname' => $data['companyname'] ?? '',
            'address1' => $data['address1'] ?? '',
            'address2' => $data['address2'] ?? '',
            'city' => $data['city'] ?? '',
            'state' => $data['state'] ?? '',
            'postcode' => $data['postcode'] ?? '',
            'country' => $data['country'] ?? 'US',
            'phonenumber' => $data['phonenumber'] ?? '',
            'password' => $this->generatePassword(),
            'datecreated' => Carbon::now()->toDateTimeString(),
            'language' => $data['language'] ?? '',
            'notes' => $data['notes'] ?? '',
            'status' => 'Active'
        ];
        
        $userId = Capsule::table('tblclients')->insertGetId($clientData);
        
        // Send welcome email if enabled
        if ($options['send_welcome']) {
            $this->sendWelcomeEmail($userId, $clientData);
        }
        
        return $userId;
    }
    
    /**
     * Update existing client
     */
    private function updateClient($userId, $data) {
        $updateData = [];
        
        $fields = ['firstname', 'lastname', 'companyname', 'address1', 'address2', 
                   'city', 'state', 'postcode', 'country', 'phonenumber'];
        
        foreach ($fields as $field) {
            if (isset($data[$field])) {
                $updateData[$field] = trim($data[$field]);
            }
        }
        
        if (!empty($updateData)) {
            Capsule::table('tblclients')
                ->where('id', $userId)
                ->update($updateData);
        }
        
        return $userId;
    }
    
    /**
     * Import services/hosting accounts
     */
    public function importServices($filePath, $options = []) {
        $defaults = [
            'delimiter' => ',',
            'has_header' => true,
            'create_missing_clients' => false
        ];
        $options = array_merge($defaults, $options);
        
        // Implementation similar to importClients
        // Maps client_email to existing client ID
        // Creates service with product information
    }
    
    /**
     * Import domains
     */
    public function importDomains($filePath, $options = []) {
        // Implementation for domain imports
    }
    
    /**
     * Generate random password
     */
    private function generatePassword($length = 12) {
        $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%';
        return substr(str_shuffle($chars), 0, $length);
    }
    
    /**
     * Send welcome email
     */
    private function sendWelcomeEmail($userId, $clientData) {
        sendEmail('Welcome Email', $userId, [
            'client_password' => $clientData['password']
        ]);
    }
}
```

### Step 3: Execute Import

```php
<?php
/**
 * Execute import via admin or CLI
 */
add_hook('AdminAreaPage', 1, function($vars) {
    if ($vars['filename'] === 'import') {
        return $vars; // Handle import form
    }
});

// CLI execution
if (php_sapi_name() === 'cli' && isset($argv[1])) {
    $import = new DataImport();
    
    $type = $argv[1]; // clients, services, domains
    $file = $argv[2];  // path to CSV file
    
    switch ($type) {
        case 'clients':
            $result = $import->importClients($file, [
                'update_existing' => true,
                'send_welcome' => false
            ]);
            break;
        case 'services':
            $result = $import->importServices($file);
            break;
    }
    
    echo "Import Complete\n";
    echo "Imported: " . $result['imported'] . "\n";
    echo "Skipped: " . $result['skipped'] . "\n";
    
    if (!empty($result['errors'])) {
        echo "Errors:\n";
        foreach ($result['errors'] as $error) {
            echo "  - $error\n";
        }
    }
}
```

### Step 4: Validate Import Data

```php
<?php
/**
 * Validate import data before processing
 */
class ImportValidator {
    
    public function validateFile($filePath, $type) {
        $errors = [];
        
        // Check file exists
        if (!file_exists($filePath)) {
            return ['valid' => false, 'errors' => ['File not found']];
        }
        
        // Check file size
        if (filesize($filePath) > 50 * 1024 * 1024) { // 50MB limit
            $errors[] = 'File size exceeds 50MB limit';
        }
        
        // Validate file extension
        $ext = strtolower(pathinfo($filePath, PATHINFO_EXTENSION));
        if (!in_array($ext, ['csv', 'xml', 'json'])) {
            $errors[] = 'Invalid file type. Allowed: CSV, XML, JSON';
        }
        
        // Type-specific validation
        switch ($type) {
            case 'clients':
                $errors = array_merge($errors, $this->validateClientFields($filePath));
                break;
            case 'services':
                $errors = array_merge($errors, $this->validateServiceFields($filePath));
                break;
        }
        
        return [
            'valid' => empty($errors),
            'errors' => $errors
        ];
    }
    
    private function validateClientFields($filePath) {
        $errors = [];
        
        $handle = fopen($filePath, 'r');
        $headers = fgetcsv($handle);
        fclose($handle);
        
        $required = ['firstname', 'lastname', 'email'];
        $headers = array_map('strtolower', array_map('trim', $headers));
        
        foreach ($required as $field) {
            if (!in_array($field, $headers)) {
                $errors[] = "Missing required column: $field";
            }
        }
        
        return $errors;
    }
    
    private function validateServiceFields($filePath) {
        $errors = [];
        // Similar validation for service fields
        return $errors;
    }
}
```

### Step 5: Import Logging

```php
<?php
/**
 * Log all import activities
 */
class ImportLogger {
    
    private $logTable = 'mod_import_logs';
    
    public function __construct() {
        $this->ensureTable();
    }
    
    private function ensureTable() {
        if (!Capsule::schema()->hasTable($this->logTable)) {
            Capsule::schema()->create($this->logTable, function($table) {
                $table->increments('id');
                $table->string('type', 50);
                $table->string('filename', 255);
                $table->integer('total_rows');
                $table->integer('imported');
                $table->integer('skipped');
                $table->text('errors')->nullable();
                $table->string('admin_id')->nullable();
                $table->timestamp('created_at');
            });
        }
    }
    
    public function log($type, $filename, $result, $adminId = null) {
        Capsule::table($this->logTable)->insert([
            'type' => $type,
            'filename' => $filename,
            'total_rows' => $result['imported'] + $result['skipped'],
            'imported' => $result['imported'],
            'skipped' => $result['skipped'],
            'errors' => json_encode($result['errors']),
            'admin_id' => $adminId,
            'created_at' => Carbon::now()->toDateTimeString()
        ]);
    }
    
    public function getHistory($limit = 50) {
        return Capsule::table($this->logTable)
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get();
    }
}
```

## Best Practices
- Always backup before importing
- Test import on staging first
- Use UTF-8 encoding
- Validate data before importing
- Handle duplicates appropriately
- Log all import activities
- Use transactions for data integrity
- Process large files in chunks
- Provide clear error messages
- Document import procedures
