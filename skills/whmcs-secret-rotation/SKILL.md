# WHMCS Secret Rotation Skill

## Purpose
Provides patterns for implementing secret rotation in WHMCS, managing API keys, credentials, and sensitive configuration with automated rotation.

## Implementation Patterns

### Secret Rotation Manager
```php
<?php
class SecretRotationManager {
    private $db;
    
    public function registerSecret($data) {
        $secret = [
            'name' => $data['name'],
            'type' => $data['type'],
            'encrypted_value' => $this->encrypt($data['value']),
            'version' => 1,
            'rotation_interval_days' => $data['rotation_interval'],
            'last_rotated_at' => date('Y-m-d H:i:s'),
            'next_rotation_at' => date('Y-m-d', strtotime("+{$data['rotation_interval']} days")),
            'status' => 'active'
        ];
        
        return $this->db->insert('mod_secrets', $secret);
    }
    
    public function rotateSecret($secretId) {
        $secret = $this->getSecret($secretId);
        
        // Generate new value
        $newValue = $this->generateSecret($secret['type']);
        
        // Store previous version
        $this->storeVersion($secretId, $secret['encrypted_value'], $secret['version']);
        
        // Update current version
        $this->db->where('id', $secretId)->update('mod_secrets', [
            'encrypted_value' => $this->encrypt($newValue),
            'version' => $secret['version'] + 1,
            'last_rotated_at' => date('Y-m-d H:i:s'),
            'next_rotation_at' => date('Y-m-d', strtotime("+{$secret['rotation_interval_days']} days"))
        ]);
        
        // Execute rotation hooks
        $this->executeRotationHooks($secretId, $newValue);
        
        logActivity("Secret rotated: {$secret['name']}");
        
        return $newValue;
    }
    
    public function getSecret($secretId) {
        $secret = $this->db->select(
            "SELECT * FROM mod_secrets WHERE id = ?",
            [$secretId]
        );
        
        $secret['value'] = $this->decrypt($secret['encrypted_value']);
        
        return $secret;
    }
    
    public function checkRotationDue() {
        $due = $this->db->select(
            "SELECT * FROM mod_secrets 
             WHERE next_rotation_at <= CURDATE() AND status = 'active'"
        );
        
        foreach ($due as $secret) {
            $this->rotateSecret($secret['id']);
        }
        
        return count($due);
    }
    
    private function generateSecret($type) {
        $lengths = ['api_key' => 32, 'password' => 24, 'token' => 64];
        $length = $lengths[$type] ?? 32;
        
        return bin2hex(random_bytes($length / 2));
    }
    
    private function encrypt($value) {
        return \Illuminate\Support\Facades\Crypt::encryptString($value);
    }
    
    private function decrypt($value) {
        return \Illuminate\Support\Facades\Crypt::decryptString($value);
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_secrets (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    type VARCHAR(50),
    encrypted_value TEXT,
    version INT,
    rotation_interval_days INT,
    last_rotated_at DATETIME,
    next_rotation_at DATE,
    status ENUM('active', 'rotating', 'inactive') DEFAULT 'active'
);

CREATE TABLE mod_secret_versions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    secret_id INT,
    encrypted_value TEXT,
    version INT,
    rotated_at DATETIME
);
```

## Usage Examples
```php
$secrets = new SecretRotationManager();
$secrets->registerSecret([
    'name' => 'payment_api_key',
    'type' => 'api_key',
    'value' => 'current_key_value',
    'rotation_interval' => 90
]);

// Check and rotate due secrets
$rotated = $secrets->checkRotationDue();
```
