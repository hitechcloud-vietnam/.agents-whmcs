# WHMCS Data Decryption Workflow

## Purpose
Decrypt encrypted data in WHMCS securely.

## Prerequisites
- WHMCS installation
- Encryption key available
- Admin access

## Step-by-Step Process

### Step 1: Create Decryption Handler

**Create hooks/data_decryption.php:**
```php
<?php
/**
 * WHMCS Data Decryption System
 */

class DataDecryption {
    
    private $encryption;
    
    public function __construct() {
        $this->encryption = new DataEncryption();
    }
    
    /**
     * Decrypt client data
     */
    public function decryptClientData($clientId) {
        $client = Capsule::table('tblclients')->where('id', $clientId)->first();
        
        if (empty($client->encrypted_data)) {
            return null;
        }
        
        return $this->encryption->decrypt($client->encrypted_data);
    }
    
    /**
     * Batch decrypt clients
     */
    public function batchDecrypt($clientIds) {
        $results = [];
        
        foreach ($clientIds as $id) {
            try {
                $results[$id] = $this->decryptClientData($id);
            } catch (Exception $e) {
                $results[$id] = ['error' => $e->getMessage()];
            }
        }
        
        return $results;
    }
}
```

### Step 2: Execute Decryption

```php
<?php
$decryption = new DataDecryption();

// Decrypt single client
$data = $decryption->decryptClientData(123);
print_r($data);

// Batch decrypt
$results = $decryption->batchDecrypt([123, 456, 789]);
```

## Best Practices
- Log all decryption access
- Time-limited access
- Secure environment
- Audit trail
- Minimize exposure
