# WHMCS XML Import Workflow

## Purpose
Import data into WHMCS using XML format with validation.

## Prerequisites
- WHMCS installation
- XML file prepared
- Admin access

## Step-by-Step Process

### Step 1: XML Format Requirements

**Client XML Structure:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<WHMCSImport version="1.0">
    <Clients>
        <Client>
            <FirstName>John</FirstName>
            <LastName>Smith</LastName>
            <Email>john@example.com</Email>
            <CompanyName>Acme Inc</CompanyName>
            <Address1>123 Main St</Address1>
            <Address2>Suite 100</Address2>
            <City>New York</City>
            <State>NY</State>
            <Postcode>10001</Postcode>
            <Country>US</Country>
            <PhoneNumber>555-1234</PhoneNumber>
            <Password>changeme123</Password>
        </Client>
    </Clients>
</WHMCSImport>
```

### Step 2: Create XML Import Handler

**Create hooks/xml_import.php:**
```php
<?php
/**
 * WHMCS XML Import Handler
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class XMLImport {
    
    private $errors = [];
    private $imported = 0;
    private $skipped = 0;
    
    /**
     * Parse XML file
     */
    public function parseXML($filePath) {
        if (!file_exists($filePath)) {
            throw new Exception("File not found: $filePath");
        }
        
        $xmlContent = file_get_contents($filePath);
        
        // Suppress XML warnings and convert to exceptions
        libxml_use_internal_errors(true);
        
        $xml = simplexml_load_string($xmlContent);
        
        if ($xml === false) {
            $errors = libxml_get_errors();
            libxml_clear_errors();
            throw new Exception("Invalid XML: " . $errors[0]->message);
        }
        
        return $xml;
    }
    
    /**
     * Import from XML object
     */
    public function importFromXML($xml, $options = []) {
        $defaults = [
            'update_existing' => false,
            'send_welcome' => false
        ];
        $options = array_merge($defaults, $options);
        
        // Import clients
        if (isset($xml->Clients)) {
            $this->importClients($xml->Clients, $options);
        }
        
        // Import services
        if (isset($xml->Services)) {
            $this->importServices($xml->Services, $options);
        }
        
        // Import domains
        if (isset($xml->Domains)) {
            $this->importDomains($xml->Domains, $options);
        }
        
        // Import products
        if (isset($xml->Products)) {
            $this->importProducts($xml->Products, $options);
        }
        
        return $this->getImportResults();
    }
    
    /**
     * Import clients
     */
    public function importClients($clients, $options) {
        foreach ($clients->Client as $clientXml) {
            try {
                $this->importClient($clientXml, $options);
                $this->imported++;
            } catch (Exception $e) {
                $this->errors[] = $e->getMessage();
                $this->skipped++;
            }
        }
    }
    
    /**
     * Import single client
     */
    private function importClient($clientXml, $options) {
        $email = strtolower((string)$clientXml->Email);
        
        // Validate required fields
        if (empty($email)) {
            throw new Exception("Email is required");
        }
        
        if (empty($clientXml->FirstName) || empty($clientXml->LastName)) {
            throw new Exception("First name and last name are required");
        }
        
        // Check existing
        $existing = Capsule::table('tblclients')
            ->where('email', $email)
            ->first();
        
        if ($existing) {
            if ($options['update_existing']) {
                $this->updateClient($existing->id, $clientXml);
                return;
            } else {
                throw new Exception("Client already exists: $email");
            }
        }
        
        // Create client
        $data = [
            'uuid' => Capsule::raw('UUID()'),
            'firstname' => (string)$clientXml->FirstName,
            'lastname' => (string)$clientXml->LastName,
            'email' => $email,
            'companyname' => (string)($clientXml->CompanyName ?? ''),
            'address1' => (string)($clientXml->Address1 ?? ''),
            'address2' => (string)($clientXml->Address2 ?? ''),
            'city' => (string)($clientXml->City ?? ''),
            'state' => (string)($clientXml->State ?? ''),
            'postcode' => (string)($clientXml->Postcode ?? ''),
            'country' => strtoupper((string)($clientXml->Country ?? 'US')),
            'phonenumber' => (string)($clientXml->PhoneNumber ?? ''),
            'password' => (string)($clientXml->Password ?? $this->generatePassword()),
            'datecreated' => $this->parseDate((string)($clientXml->DateCreated ?? 'now')),
            'language' => (string)($clientXml->Language ?? ''),
            'notes' => (string)($clientXml->Notes ?? ''),
            'status' => (string)($clientXml->Status ?? 'Active')
        ];
        
        $userId = Capsule::table('tblclients')->insertGetId($data);
        
        if ($options['send_welcome']) {
            $this->sendWelcomeEmail($userId, $data);
        }
    }
    
    /**
     * Update existing client
     */
    private function updateClient($userId, $clientXml) {
        $updateData = [];
        
        $fields = ['firstname', 'lastname', 'companyname', 'address1', 'address2',
                   'city', 'state', 'postcode', 'country', 'phonenumber'];
        
        foreach ($fields as $field) {
            $xmlField = ucfirst($field);
            if (isset($clientXml->$xmlField)) {
                $updateData[$field] = (string)$clientXml->$xmlField;
            }
        }
        
        if (!empty($updateData)) {
            Capsule::table('tblclients')
                ->where('id', $userId)
                ->update($updateData);
        }
    }
    
    /**
     * Import services
     */
    public function importServices($services, $options) {
        foreach ($services->Service as $serviceXml) {
            try {
                $this->importService($serviceXml, $options);
                $this->imported++;
            } catch (Exception $e) {
                $this->errors[] = $e->getMessage();
                $this->skipped++;
            }
        }
    }
    
    /**
     * Import single service
     */
    private function importService($serviceXml, $options) {
        $email = strtolower((string)$serviceXml->ClientEmail);
        $productName = (string)$serviceXml->ProductName;
        $domain = (string)($serviceXml->Domain ?? '');
        
        // Get client
        $client = Capsule::table('tblclients')
            ->where('email', $email)
            ->first();
        
        if (!$client) {
            throw new Exception("Client not found: $email");
        }
        
        // Get product
        $product = Capsule::table('tblproducts')
            ->where('name', $productName)
            ->first();
        
        if (!$product) {
            throw new Exception("Product not found: $productName");
        }
        
        // Check existing service
        $existing = Capsule::table('tblhosting')
            ->where('userid', $client->id)
            ->where('domain', $domain)
            ->first();
        
        if ($existing && !$options['update_existing']) {
            throw new Exception("Service already exists for $domain");
        }
        
        $data = [
            'userid' => $client->id,
            'packageid' => $product->id,
            'server' => (int)($serviceXml->ServerID ?? 0),
            'regdate' => $this->parseDate((string)($serviceXml->RegDate ?? 'now')),
            'domain' => $domain,
            'firstpaymentamount' => (float)($serviceXml->FirstPayment ?? 0),
            'recurringamount' => (float)($serviceXml->RecurringAmount ?? $product->recurring),
            'billingcycle' => (string)($serviceXml->BillingCycle ?? 'Monthly'),
            'nextduedate' => $this->parseDate((string)($serviceXml->NextDueDate ?? 'now')),
            'domainstatus' => (string)($serviceXml->Status ?? 'Active'),
            'username' => (string)($serviceXml->Username ?? ''),
            'password' => (string)($serviceXml->Password ?? ''),
            'notes' => (string)($serviceXml->Notes ?? '')
        ];
        
        if ($existing) {
            Capsule::table('tblhosting')
                ->where('id', $existing->id)
                ->update($data);
        } else {
            Capsule::table('tblhosting')->insert($data);
        }
    }
    
    /**
     * Import domains
     */
    public function importDomains($domains, $options) {
        foreach ($domains->Domain as $domainXml) {
            try {
                $this->importDomain($domainXml, $options);
                $this->imported++;
            } catch (Exception $e) {
                $this->errors[] = $e->getMessage();
                $this->skipped++;
            }
        }
    }
    
    /**
     * Import single domain
     */
    private function importDomain($domainXml, $options) {
        $email = strtolower((string)$domainXml->ClientEmail);
        $domainName = (string)$domainXml->DomainName;
        
        // Get client
        $client = Capsule::table('tblclients')
            ->where('email', $email)
            ->first();
        
        if (!$client) {
            throw new Exception("Client not found: $email");
        }
        
        // Get registrar
        $registrar = (string)($domainXml->Registrar ?? 'enom');
        
        // Check existing
        $existing = Capsule::table('tbldomains')
            ->where('userid', $client->id)
            ->where('domain', $domainName)
            ->first();
        
        if ($existing && !$options['update_existing']) {
            throw new Exception("Domain already exists: $domainName");
        }
        
        $data = [
            'userid' => $client->id,
            'domain' => $domainName,
            'registrationperiod' => (int)($domainXml->RegistrationPeriod ?? 1),
            'registrar' => $registrar,
            'regdate' => $this->parseDate((string)($domainXml->RegDate ?? 'now')),
            'domainstatus' => (string)($domainXml->Status ?? 'Active'),
            'nextduedate' => $this->parseDate((string)($domainXml->NextDueDate ?? 'now')),
            'expirydate' => $this->parseDate((string)($domainXml->ExpiryDate ?? 'now')),
            'dnsmanagement' => (int)($domainXml->DNSManagement ?? 0),
            'emailforwarding' => (int)($domainXml->EmailForwarding ?? 0),
            'idprotection' => (int)($domainXml->IDProtection ?? 0),
            'registrarorderid' => (string)($domainXml->RegistrarOrderID ?? ''),
            'registraramount' => (float)($domainXml->RegistrationAmount ?? 0),
            'recurringamount' => (float)($domainXml->RenewalAmount ?? 0)
        ];
        
        if ($existing) {
            Capsule::table('tbldomains')
                ->where('id', $existing->id)
                ->update($data);
        } else {
            Capsule::table('tbldomains')->insert($data);
        }
    }
    
    /**
     * Parse date string
     */
    private function parseDate($dateStr) {
        if ($dateStr === 'now' || empty($dateStr)) {
            return Carbon::now()->toDateString();
        }
        
        try {
            return Carbon::parse($dateStr)->toDateString();
        } catch (Exception $e) {
            return Carbon::now()->toDateString();
        }
    }
    
    /**
     * Generate password
     */
    private function generatePassword($length = 12) {
        $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%';
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
            'skipped' => $this->skipped,
            'errors' => $this->errors
        ];
    }
}
```

### Step 3: Execute XML Import

```php
<?php
/**
 * Execute XML import
 */
$importer = new XMLImport();

try {
    $xml = $importer->parseXML('/path/to/import.xml');
    
    $result = $importer->importFromXML($xml, [
        'update_existing' => true,
        'send_welcome' => false
    ]);
    
    echo "Import Complete\n";
    echo "Imported: " . $result['imported'] . "\n";
    echo "Skipped: " . $result['skipped'] . "\n";
    
    if (!empty($result['errors'])) {
        echo "\nErrors:\n";
        foreach ($result['errors'] as $error) {
            echo "- $error\n";
        }
    }
    
} catch (Exception $e) {
    echo "Import failed: " . $e->getMessage() . "\n";
}
```

## Best Practices
- Always validate XML before import
- Use well-formed XML
- Test on small dataset first
- Backup before importing
- Handle duplicates appropriately
- Log all import activities
- Use UTF-8 encoding
- Verify required fields
- Document XML schema
- Use transactions for integrity
