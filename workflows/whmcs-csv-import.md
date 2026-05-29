# WHMCS CSV Import Workflow

## Purpose
Import data into WHMCS using CSV files with validation and error handling.

## Prerequisites
- WHMCS installation
- CSV file prepared
- Admin access

## Step-by-Step Process

### Step 1: Prepare CSV File

**File Requirements:**
```
- Format: UTF-8 encoded
- Delimiter: Comma (,) or Semicolon (;)
- Enclosure: Double quotes (")
- Line endings: LF or CRLF
```

**Client Import Template:**
```csv
firstname,lastname,email,companyname,address1,address2,city,state,postcode,country,phonenumber
John,Smith,john@example.com,Acme Inc,123 Main St,Suite 100,New York,NY,10001,US,555-1234
Jane,Doe,jane@example.com,,456 Oak Ave,Boston,MA,02101,US,555-5678
```

### Step 2: Create CSV Import Handler

**Create hooks/csv_import.php:**
```php
<?php
/**
 * WHMCS CSV Import Handler
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class CSVImport {
    
    private $requiredFields = ['firstname', 'lastname', 'email'];
    private $errors = [];
    private $warnings = [];
    private $imported = 0;
    private $skipped = 0;
    private $updated = 0;
    
    /**
     * Parse CSV file
     */
    public function parseCSV($filePath, $options = []) {
        $defaults = [
            'delimiter' => ',',
            'enclosure' => '"',
            'escape' => '"',
            'has_header' => true,
            'skip_empty_rows' => true
        ];
        $options = array_merge($defaults, $options);
        
        if (!file_exists($filePath)) {
            throw new Exception("File not found: $filePath");
        }
        
        $handle = fopen($filePath, 'r');
        if (!$handle) {
            throw new Exception("Cannot open file: $filePath");
        }
        
        $data = [];
        $headers = [];
        $rowNum = 0;
        
        while (($row = fgetcsv($handle, 0, $options['delimiter'], $options['enclosure'], $options['escape'])) !== false) {
            $rowNum++;
            
            // Skip header row
            if ($rowNum === 1 && $options['has_header']) {
                $headers = array_map('strtolower', array_map('trim', $row));
                continue;
            }
            
            // Skip empty rows
            if ($options['skip_empty_rows'] && empty(array_filter($row))) {
                continue;
            }
            
            // Map to headers
            if (!empty($headers)) {
                $data[] = array_combine($headers, $row);
            } else {
                $data[] = $row;
            }
        }
        
        fclose($handle);
        
        return [
            'headers' => $headers,
            'data' => $data,
            'total_rows' => count($data)
        ];
    }
    
    /**
     * Validate CSV data
     */
    public function validateData($data, $entityType) {
        $errors = [];
        $rowNum = 0;
        
        foreach ($data as $row) {
            $rowNum++;
            
            switch ($entityType) {
                case 'clients':
                    $rowErrors = $this->validateClientRow($row, $rowNum);
                    break;
                case 'services':
                    $rowErrors = $this->validateServiceRow($row, $rowNum);
                    break;
                case 'domains':
                    $rowErrors = $this->validateDomainRow($row, $rowNum);
                    break;
                default:
                    $rowErrors = [];
            }
            
            $errors = array_merge($errors, $rowErrors);
        }
        
        return [
            'valid' => empty($errors),
            'errors' => $errors
        ];
    }
    
    /**
     * Validate client row
     */
    private function validateClientRow($row, $rowNum) {
        $errors = [];
        
        // Required fields
        foreach (['firstname', 'lastname', 'email'] as $field) {
            if (empty($row[$field])) {
                $errors[] = "Row $rowNum: Missing required field '$field'";
            }
        }
        
        // Email validation
        if (!empty($row['email']) && !filter_var($row['email'], FILTER_VALIDATE_EMAIL)) {
            $errors[] = "Row $rowNum: Invalid email format";
        }
        
        // Country validation
        if (!empty($row['country'])) {
            $validCountries = $this->getValidCountries();
            if (!in_array(strtoupper($row['country']), $validCountries)) {
                $errors[] = "Row $rowNum: Invalid country code";
            }
        }
        
        return $errors;
    }
    
    /**
     * Validate service row
     */
    private function validateServiceRow($row, $rowNum) {
        $errors = [];
        
        // Client email required
        if (empty($row['client_email']) && empty($row['userid'])) {
            $errors[] = "Row $rowNum: Client email or ID required";
        }
        
        // Product name required
        if (empty($row['product_name']) && empty($row['packageid'])) {
            $errors[] = "Row $rowNum: Product name or ID required";
        }
        
        return $errors;
    }
    
    /**
     * Validate domain row
     */
    private function validateDomainRow($row, $rowNum) {
        $errors = [];
        
        // Domain required
        if (empty($row['domain'])) {
            $errors[] = "Row $rowNum: Domain name required";
        } elseif (!filter_var('http://' . $row['domain'], FILTER_VALIDATE_URL) && 
                   !preg_match('/^[a-zA-Z0-9][a-zA-Z0-9-]*\.[a-zA-Z]{2,}$/', $row['domain'])) {
            $errors[] = "Row $rowNum: Invalid domain format";
        }
        
        return $errors;
    }
    
    /**
     * Import clients from validated data
     */
    public function importClients($data, $options = []) {
        $defaults = [
            'update_existing' => false,
            'send_welcome' => false,
            'default_password' => null
        ];
        $options = array_merge($defaults, $options);
        
        foreach ($data as $row) {
            try {
                $this->importClient($row, $options);
                $this->imported++;
            } catch (Exception $e) {
                $this->errors[] = $e->getMessage();
                $this->skipped++;
            }
        }
        
        return $this->getImportResults();
    }
    
    /**
     * Import single client
     */
    private function importClient($row, $options) {
        $email = strtolower(trim($row['email']));
        
        // Check existing
        $existing = Capsule::table('tblclients')
            ->where('email', $email)
            ->first();
        
        if ($existing) {
            if ($options['update_existing']) {
                $this->updateExistingClient($existing->id, $row);
                $this->updated++;
                return;
            } else {
                throw new Exception("Client already exists: $email");
            }
        }
        
        // Create new client
        $clientData = $this->prepareClientData($row, $options);
        $userId = Capsule::table('tblclients')->insertGetId($clientData);
        
        // Send welcome email
        if ($options['send_welcome']) {
            $this->sendWelcomeEmail($userId, $clientData);
        }
    }
    
    /**
     * Prepare client data array
     */
    private function prepareClientData($row, $options) {
        return [
            'uuid' => Capsule::raw('UUID()'),
            'firstname' => trim($row['firstname']),
            'lastname' => trim($row['lastname']),
            'email' => strtolower(trim($row['email'])),
            'companyname' => trim($row['companyname'] ?? ''),
            'address1' => trim($row['address1'] ?? ''),
            'address2' => trim($row['address2'] ?? ''),
            'city' => trim($row['city'] ?? ''),
            'state' => trim($row['state'] ?? ''),
            'postcode' => trim($row['postcode'] ?? ''),
            'country' => strtoupper(trim($row['country'] ?? 'US')),
            'phonenumber' => trim($row['phonenumber'] ?? ''),
            'password' => $options['default_password'] ?? $this->generatePassword(),
            'datecreated' => Carbon::now()->toDateTimeString(),
            'language' => trim($row['language'] ?? ''),
            'notes' => trim($row['notes'] ?? ''),
            'status' => 'Active'
        ];
    }
    
    /**
     * Update existing client
     */
    private function updateExistingClient($userId, $row) {
        $updateFields = ['firstname', 'lastname', 'companyname', 'address1', 'address2',
                        'city', 'state', 'postcode', 'country', 'phonenumber'];
        
        $updateData = [];
        foreach ($updateFields as $field) {
            if (isset($row[$field])) {
                $updateData[$field] = trim($row[$field]);
            }
        }
        
        if (!empty($updateData)) {
            Capsule::table('tblclients')
                ->where('id', $userId)
                ->update($updateData);
        }
    }
    
    /**
     * Generate random password
     */
    private function generatePassword($length = 12) {
        $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*';
        return substr(str_shuffle($chars), 0, $length);
    }
    
    /**
     * Send welcome email
     */
    private function sendWelcomeEmail($userId, $data) {
        sendEmail('Welcome Email', $userId, [
            'client_password' => $data['password']
        ]);
    }
    
    /**
     * Get import results
     */
    public function getImportResults() {
        return [
            'imported' => $this->imported,
            'updated' => $this->updated,
            'skipped' => $this->skipped,
            'errors' => $this->errors,
            'warnings' => $this->warnings
        ];
    }
    
    /**
     * Get valid country codes
     */
    private function getValidCountries() {
        return [
            'US', 'CA', 'GB', 'AU', 'DE', 'FR', 'ES', 'IT', 'NL', 'BE',
            'AT', 'CH', 'IE', 'NZ', 'SG', 'HK', 'JP', 'KR', 'IN', 'CN',
            'BR', 'MX', 'AR', 'CL', 'CO', 'PE', 'VE', 'ZA', 'EG', 'NG',
            'KE', 'AE', 'SA', 'IL', 'TR', 'PL', 'CZ', 'HU', 'RO', 'RU',
            'UA', 'SE', 'NO', 'DK', 'FI', 'PT', 'GR', 'TH', 'VN', 'MY',
            'PH', 'ID', 'TW', 'OTHER'
        ];
    }
}
```

### Step 3: Execute CSV Import

```php
<?php
/**
 * Execute CSV import
 */
$importer = new CSVImport();

// Parse CSV file
$csv = $importer->parseCSV('/path/to/clients.csv', [
    'delimiter' => ',',
    'has_header' => true
]);

// Validate data
$validation = $importer->validateData($csv['data'], 'clients');

if (!$validation['valid']) {
    // Handle validation errors
    foreach ($validation['errors'] as $error) {
        echo $error . "\n";
    }
    exit;
}

// Import clients
$result = $importer->importClients($csv['data'], [
    'update_existing' => true,
    'send_welcome' => false
]);

// Output results
echo "Import Complete\n";
echo "Imported: " . $result['imported'] . "\n";
echo "Updated: " . $result['updated'] . "\n";
echo "Skipped: " . $result['skipped'] . "\n";

if (!empty($result['errors'])) {
    echo "\nErrors:\n";
    foreach ($result['errors'] as $error) {
        echo "- $error\n";
    }
}
```

### Step 4: Bulk Import Services

```php
<?php
/**
 * Import services from CSV
 */
$importer = new CSVImport();

$csv = $importer->parseCSV('/path/to/services.csv');
$validation = $importer->validateData($csv['data'], 'services');

// Import services
$result = $importer->importServices($csv['data']);
```

## Best Practices
- Always backup before import
- Use UTF-8 encoding
- Include header row
- Validate before importing
- Handle duplicates gracefully
- Log all import activities
- Test on small dataset first
- Use appropriate field sizes
- Clean up data before import
- Verify import results
