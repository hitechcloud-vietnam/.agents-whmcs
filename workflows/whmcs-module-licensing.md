# WHMCS Module Licensing Workflow

## Description
Implement license validation for commercial WHMCS modules.

## Steps

### Step 1: Create License Class
```php
<?php
/**
 * License Validation Class
 */

class CLICodesLicense
{
    private $licenseKey;
    private $domain;
    private $productId;
    private $apiUrl;
    
    public function __construct()
    {
        $this->licenseKey = ''; // From configuration
        $this->domain = $_SERVER['HTTP_HOST'];
        $this->productId = 'module_product_id';
        $this->apiUrl = 'https://license.example.com/api';
    }
    
    /**
     * Validate license
     */
    public function validate()
    {
        $result = $this->apiCall('validate', [
            'license_key' => $this->licenseKey,
            'domain' => $this->domain,
            'product_id' => $this->productId,
        ]);
        
        if ($result['status'] === 'valid') {
            $this->storeValidation($result);
            return true;
        }
        
        return false;
    }
    
    /**
     * Check if license is active
     */
    public function isActive()
    {
        $cached = $this->getCachedValidation();
        
        if ($cached && $cached['expires'] > time()) {
            return true;
        }
        
        return $this->validate();
    }
    
    /**
     * Get license information
     */
    public function getInfo()
    {
        return $this->apiCall('info', [
            'license_key' => $this->licenseKey,
        ]);
    }
    
    /**
     * API call to license server
     */
    private function apiCall($action, $data)
    {
        $ch = curl_init($this->apiUrl . '/' . $action);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
            CURLOPT_TIMEOUT => 30,
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true);
    }
    
    /**
     * Store validation in cache
     */
    private function storeValidation($data)
    {
        $data['expires'] = time() + 86400; // 24 hours
        file_put_contents(
            __DIR__ . '/.license_cache',
            json_encode($data)
        );
    }
    
    /**
     * Get cached validation
     */
    private function getCachedValidation()
    {
        $file = __DIR__ . '/.license_cache';
        
        if (file_exists($file)) {
            $data = json_decode(file_get_contents($file), true);
            return $data;
        }
        
        return null;
    }
}
```

### Step 2: Integrate License Check
```php
<?php
// In module main file

function clicodes_example_output($vars)
{
    $license = new CLICodesLicense();
    
    if (!$license->isActive()) {
        return '<div class="alert alert-danger">
            License validation failed. Please check your license key.
        </div>';
    }
    
    // Continue with module output
    // ...
}
```

### Step 3: Add License Configuration
```php
function clicodes_example_config()
{
    return [
        'FriendlyName' => ['value' => 'CLICodes Module'],
        'licenseKey' => [
            'Type' => 'text',
            'Label' => 'License Key',
            'Description' => 'Enter your license key',
        ],
    ];
}
```

### Step 4: Handle License Events
```php
<?php
// Handle license transfer
public function handleTransfer($newDomain)
{
    $result = $this->apiCall('transfer', [
        'license_key' => $this->licenseKey,
        'new_domain' => $newDomain,
    ]);
    
    return $result['success'];
}

// Handle license cancellation
public function handleCancellation()
{
    logActivity('Module license cancelled');
}
```

## License Server Features
- Domain validation
- Expiration checking
- Usage tracking
- Transfer handling
- Upgrade management

## Tags
- licensing
- security
- commercial
- validation