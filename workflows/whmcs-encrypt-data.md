# WHMCS Data Encryption Workflow

## Purpose
Encrypt sensitive data in WHMCS for security compliance.

## Prerequisites
- WHMCS installation
- PHP OpenSSL extension
- Admin access

## Step-by-Step Process

### Step 1: Create Encryption Handler

**Create hooks/data_encryption.php:**
```php
<?php
/**
 * WHMCS Data Encryption System
 */

class DataEncryption {
    
    private $key;
    private $cipher = 'aes-256-gcm';
    
    public function __construct() {
        $this->key = $this->getEncryptionKey();
    }
    
    /**
     * Get or generate encryption key
     */
    private function getEncryptionKey() {
        $keyFile = ROOTDIR . '/config/encryption_key.txt';
        
        if (file_exists($keyFile)) {
            return file_get_contents($keyFile);
        }
        
        // Generate new key
        $key = bin2hex(random_bytes(32));
        file_put_contents($keyFile, $key);
        chmod($keyFile, 0600);
        
        return $key;
    }
    
    /**
     * Encrypt data
     */
    public function encrypt($data) {
        if (is_array($data)) {
            $data = json_encode($data);
        }
        
        $iv = random_bytes(openssl_cipher_iv_length($this->cipher));
        $tag = '';
        
        $encrypted = openssl_encrypt(
            $data,
            $this->cipher,
            hex2bin($this->key),
            OPENSSL_RAW_DATA,
            $iv,
            $tag,
            '',
            16
        );
        
        // Return IV + Tag + Encrypted data
        return base64_encode($iv . $tag . $encrypted);
    }
    
    /**
     * Decrypt data
     */
    public function decrypt($encryptedData) {
        $data = base64_decode($encryptedData);
        
        $ivLength = openssl_cipher_iv_length($this->cipher);
        $iv = substr($data, 0, $ivLength);
        $tag = substr($data, $ivLength, 16);
        $encrypted = substr($data, $ivLength + 16);
        
        $decrypted = openssl_decrypt(
            $encrypted,
            $this->cipher,
            hex2bin($this->key),
            OPENSSL_RAW_DATA,
            $iv,
            $tag
        );
        
        // Try JSON decode
        $json = json_decode($decrypted, true);
        return $json ?? $decrypted;
    }
    
    /**
     * Encrypt sensitive client fields
     */
    public function encryptClientData($clientId) {
        $client = Capsule::table('tblclients')->where('id', $clientId)->first();
        
        $encrypted = $this->encrypt([
            'email' => $client->email,
            'phonenumber' => $client->phonenumber,
            'address1' => $client->address1,
            'address2' => $client->address2,
            'city' => $client->city,
            'state' => $client->state,
            'postcode' => $client->postcode
        ]);
        
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update([
                'encrypted_data' => $encrypted
            ]);
        
        return true;
    }
}
```

### Step 2: Execute Encryption

```php
<?php
$encryption = new DataEncryption();

// Encrypt data
$encrypted = $encryption->encrypt('sensitive data');

// Decrypt data
$decrypted = $encryption->decrypt($encrypted);
```

## Best Practices
- Secure key storage
- Regular key rotation
- Secure transmission
- Audit encryption usage
- Test decryption
