---
name: whmcs-secrets-management
description: Secrets handling for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS Secrets Management Skill

## Overview
This skill provides patterns and implementations for managing secrets in WHMCS, including encryption, storage, and secure retrieval.

## Implementation Patterns

### Secrets Manager
```php
<?php
/**
 * WHMCS Secrets Management
 * Manages sensitive data securely
 */

namespace WHMCS\Module\DevOps\Secrets;

class SecretsManager {
    private $db;
    private $encryptionKey;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->encryptionKey = $this->getEncryptionKey();
    }

    /**
     * Store secret
     */
    public function storeSecret(array $params): array {
        $secretId = 'sec_' . bin2hex(random_bytes(12));

        // Encrypt the secret
        $encryptedValue = $this->encrypt($params['value']);

        $secret = [
            'id' => $secretId,
            'service_id' => $params['service_id'] ?? null,
            'name' => $params['name'],
            'encrypted_value' => $encryptedValue,
            'algorithm' => 'AES-256-GCM',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_secrets', $secret);

        return [
            'success' => true,
            'secret_id' => $secretId
        ];
    }

    /**
     * Retrieve secret
     */
    public function retrieveSecret(string $secretId): ?string {
        $secret = $this->db->select(
            "SELECT * FROM mod_secrets WHERE id = ?",
            [$secretId]
        )[0];

        if (!$secret) {
            return null;
        }

        // Log access (audit)
        $this->logAccess($secretId);

        // Decrypt and return
        return $this->decrypt($secret->encrypted_value);
    }

    /**
     * Generate API credentials
     */
    public function generateCredentials(string $serviceId): array {
        $apiKey = bin2hex(random_bytes(32));
        $apiSecret = bin2hex(random_bytes(64));

        $this->storeSecret([
            'service_id' => $serviceId,
            'name' => 'api_key',
            'value' => $apiKey
        ]);

        $this->storeSecret([
            'service_id' => $serviceId,
            'name' => 'api_secret',
            'value' => $apiSecret
        ]);

        return [
            'api_key' => $apiKey,
            'api_secret' => $apiSecret
        ];
    }

    /**
     * Rotate secret
     */
    public function rotateSecret(string $secretId): array {
        $secret = $this->db->select(
            "SELECT * FROM mod_secrets WHERE id = ?",
            [$secretId]
        )[0];

        if (!$secret) {
            throw new \Exception("Secret not found");
        }

        // Generate new value
        $newValue = bin2hex(random_bytes(32));

        // Store as new version
        $newSecretId = 'sec_' . bin2hex(random_bytes(12));

        $this->db->insert('mod_secrets', [
            'id' => $newSecretId,
            'service_id' => $secret->service_id,
            'name' => $secret->name,
            'encrypted_value' => $this->encrypt($newValue),
            'algorithm' => 'AES-256-GCM',
            'previous_version' => $secretId,
            'created_at' => date('Y-m-d H:i:s')
        ]);

        // Mark old as rotated
        $this->db->update('mod_secrets', [
            'rotated_at' => date('Y-m-d H:i:s'),
            'rotated_to' => $newSecretId
        ], ['id' => $secretId]);

        return [
            'success' => true,
            'new_secret_id' => $newSecretId
        ];
    }

    // Private helper methods

    private function encrypt(string $value): string {
        $ivLength = openssl_cipher_iv_length('aes-256-gcm');
        $iv = random_bytes($ivLength);

        $tag = '';
        $encrypted = openssl_encrypt(
            $value,
            'aes-256-gcm',
            $this->encryptionKey,
            OPENSSL_RAW_DATA,
            $iv,
            $tag
        );

        // Combine IV + encrypted + tag
        return base64_encode($iv . $encrypted . $tag);
    }

    private function decrypt(string $encrypted): string {
        $data = base64_decode($encrypted);

        $ivLength = openssl_cipher_iv_length('aes-256-gcm');
        $iv = substr($data, 0, $ivLength);
        $tagLength = 16; // AES-256-GCM tag length
        $tag = substr($data, -$tagLength);
        $encrypted = substr($data, $ivLength, -$tagLength);

        return openssl_decrypt(
            $encrypted,
            'aes-256-gcm',
            $this->encryptionKey,
            OPENSSL_RAW_DATA,
            $iv,
            $tag
        );
    }

    private function logAccess(string $secretId): void {
        $this->db->insert('mod_secrets_access_log', [
            'secret_id' => $secretId,
            'accessed_at' => date('Y-m-d H:i:s'),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'unknown'
        ]);
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_secrets` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT,
  `name` VARCHAR(255) NOT NULL,
  `encrypted_value` TEXT NOT NULL,
  `algorithm` VARCHAR(50) DEFAULT 'AES-256-GCM',
  `previous_version` VARCHAR(50),
  `rotated_at` DATETIME,
  `rotated_to` VARCHAR(50),
  `created_at' DATETIME NOT NULL
);

CREATE TABLE `mod_secrets_access_log` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `secret_id` VARCHAR(50) NOT NULL,
  `accessed_at' DATETIME NOT NULL,
  `ip_address` VARCHAR(45),
  FOREIGN KEY (`secret_id`) REFERENCES `mod_secrets`(`id`)
);
```

## Best Practices

1. **Encryption**: Always encrypt secrets at rest
2. **Key Rotation**: Rotate secrets regularly
3. **Access Logging**: Track all secret access
4. **Least Privilege**: Only grant necessary access
5. **No Plain Text**: Never store secrets in plain text

## Related Skills

- whmcs-config-management
- whmcs-cicd-integration
- whmcs-gitops-workflow
- whmcs-artifact-storage