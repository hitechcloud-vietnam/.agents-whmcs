# WHMCS API Key Generation Workflow

## Purpose
Guide developers through generating and managing WHMCS API keys.

## Prerequisites
- WHMCS installation
- Admin access
- Understanding of API authentication
- Security best practices knowledge

## Steps

### Phase 1: API Key Concepts

1. Key types
   ```
   WHMCS API Keys:
   ├── Primary API Key (Account-level)
   ├── API Credentials (Staff-based)
   ├── Access Keys (Integration)
   └── Webhook Keys (Event-based)
   ```

2. Key generation methods
   ```
   Generation Methods:
   - WHMCS Admin Panel (GUI)
   - WHMCS API (Programmatic)
   - CLI Commands
   - Admin User Management
   ```

### Phase 2: Generate API Keys via Admin

1. Create new API credential
   ```
   1. Log into WHMCS Admin
   2. Navigate to: Setup > Staff Management > API Credentials
   3. Click "Add New API Credential"
   4. Select staff member or create system user
   5. Configure permissions
   6. Set IP access restrictions (optional)
   7. Click "Generate"
   8. Copy and securely store the API key
   ```

2. Credential settings
   ```
   API Credential Options:
   ├── Description/Label
   ├── Associated Staff Member
   ├── Permissions (granular)
   ├── IP Whitelist
   ├── Rate Limiting
   └── Expiration Date (optional)
   ```

### Phase 3: Programmatic Key Generation

1. Generate via API (Admin function)
   ```php
   <?php
   // Note: Direct API key generation requires admin intervention
   // This is a conceptual example
   
   function generateApiCredential($adminUserId, $permissions, $options) {
       // This function would typically be handled through admin UI
       
       $credential = [
           'staff_id' => $adminUserId,
           'description' => $options['description'] ?? 'API Credential',
           'key_hash' => generateSecureKey(32),
           'permissions' => $permissions,
           'ip_whitelist' => $options['ip_whitelist'] ?? [],
           'created_at' => date('Y-m-d H:i:s'),
           'last_used' => null
       ];
       
       return $credential;
   }
   ```

2. Generate secure key
   ```php
   function generateSecureKey($length = 32) {
       $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
       $key = '';
       $max = strlen($chars) - 1;
       
       for ($i = 0; $i < $length; $i++) {
           $key .= $chars[crypto_rand_secure(0, $max)];
       }
       
       return $key;
   }
   
   function crypto_rand_secure($min, $max) {
       $range = $max - $min;
       if ($range == 0) return $min;
      
       $log = log($range, 2);
       $bytes = (int) ($log / 8) + 1;
       $bits = (int) $log + 1;
       $filter = (1 << $bits) - 1;
       
       do {
           $rnd = hexdec(bin2hex(openssl_random_pseudo_bytes($bytes)));
           $rnd = $rnd & $filter;
       } while ($rnd >= $range);
       
       return $min + $rnd;
   }
   ```

### Phase 4: Key Management

1. Key rotation
   ```php
   // Key rotation script
   function rotateApiKey($credentialId) {
       // 1. Generate new key
       $newKey = generateSecureKey(32);
       
       // 2. Store new key hash
       $keyHash = password_hash($newKey, PASSWORD_DEFAULT);
       
       // 3. Update in database
       updateCredential($credentialId, [
           'key_hash' => $keyHash,
           'last_rotated' => date('Y-m-d H:i:s')
       ]);
       
       // 4. Invalidate old key
       // (Key becomes invalid after rotation)
       
       return $newKey;
   }
   ```

2. Key storage
   ```php
   // Secure key storage
   class SecureKeyStorage {
       private $encryptionKey;
       
       public function __construct($encKey) {
           $this->encryptionKey = $encKey;
       }
       
       public function store($key, $label) {
           // Encrypt key before storage
           $encrypted = $this->encrypt($key);
           
           // Store encrypted version
           // (Never store plaintext keys)
           saveToVault($label, $encrypted);
       }
       
       public function retrieve($label) {
           $encrypted = getFromVault($label);
           return $this->decrypt($encrypted);
       }
   }
   ```

### Phase 5: Key Permissions

1. Permission levels
   ```
   API Permission Categories:
   ├── Account Information
   ├── Client Management
   ├── Service Management
   ├── Domain Management
   ├── Billing & Invoices
   ├── Support Tickets
   ├── Orders & Cart
   └── System Administration
   ```

2. Configure permissions
   ```php
   // Permission configuration
   $permissions = [
       // Full access
       '*' => true,
       
       // Specific permissions
       'getclients' => true,
       'getclientsdetails' => true,
       'addclient' => false,
       'updateclient' => true,
       'getproducts' => true,
       'addorder' => false,  // Denied
   ];
   ```

### Phase 6: Access Key for Webhooks

1. Generate webhook access key
   ```
   Webhook Configuration:
   1. Go to: Configuration > System Settings > Webhooks
   2. Create new webhook
   3. Select events to subscribe
   4. Generate access key
   5. Configure payload URL
   6. Set retry policies
   ```

2. Webhook key configuration
   ```php
   // Webhook verification
   function verifyWebhook($payload, $signature, $secret) {
       $expectedSignature = hash_hmac('sha256', $payload, $secret);
       
       return hash_equals($expectedSignature, $signature);
   }
   
   // Webhook handler
   function handleWebhook($data, $headers) {
       $signature = $headers['X-WHMCS-Signature'] ?? '';
       
       if (!verifyWebhook(json_encode($data), $signature, WHMCS_WEBHOOK_SECRET)) {
           http_response_code(401);
           die('Invalid webhook signature');
       }
       
       // Process webhook
       processWebhookEvent($data);
   }
   ```

### Phase 7: Key Rotation Schedule

1. Rotation policy
   ```
   Key Rotation Schedule:
   ├── Monthly: High-security keys
   ├── Quarterly: Standard API keys
   ├── On compromise: Immediate rotation
   └── On staff change: Update relevant keys
   ```

2. Automated rotation
   ```php
   // Automated rotation script (cron job)
   function checkKeyExpiration() {
       $keys = getAllApiKeys();
       
       foreach ($keys as $key) {
           $daysUntilExpiry = daysUntil($key['expires_at']);
           
           if ($daysUntilExpiry <= 7) {
               // Send expiration warning
               sendNotification($key['owner_email'], 
                   "API Key expires in $daysUntilExpiry days");
           }
           
           if ($daysUntilExpiry <= 0) {
               // Auto-rotate key
               rotateKey($key['id']);
           }
       }
   }
   ```

### Phase 8: Key Documentation

1. API key documentation
   ```markdown
   # WHMCS API Key Management
   
   ## Generated Keys
   
   | Key Name | Created | Expires | Permissions |
   |----------|---------|---------|-------------|
   | Integration-A | 2024-01-15 | 2024-07-15 | Read-only |
   | Integration-B | 2024-02-20 | 2024-08-20 | Full |
   
   ## Usage Guidelines
   
   - Store keys securely (use vault)
   - Rotate keys regularly
   - Monitor key usage
   - Revoke unused keys
   ```

## Related Workflows
- whmcs-api-authentication
- whmcs-api-webhook-setup
- whmcs-api-integration
- whmcs-api-security