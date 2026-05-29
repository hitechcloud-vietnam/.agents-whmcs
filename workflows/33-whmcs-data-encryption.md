# WHMCS Data Encryption Workflow

## Overview
This workflow covers implementing and managing data encryption for sensitive information in WHMCS.

## Step 1: Encryption Service

```php
<?php
// src/Service/EncryptionService.php

namespace WHMCS\Module\Addon\YourModule\Service;

class EncryptionService
{
    private $cipher = 'aes-256-gcm';
    private $key;

    public function __construct(string $encryptionKey = null)
    {
        $this->key = $encryptionKey ?? $this->getOrCreateKey();
    }

    public function encrypt(string $data): string
    {
        $iv = random_bytes(openssl_cipher_iv_length($this->cipher));
        $tag = '';

        $encrypted = openssl_encrypt(
            $data,
            $this->cipher,
            $this->key,
            OPENSSL_RAW_DATA,
            $iv,
            $tag,
            '',
            16
        );

        // Return IV + Tag + Ciphertext
        return base64_encode($iv . $tag . $encrypted);
    }

    public function decrypt(string $data): string
    {
        $data = base64_decode($data);
        $ivLength = openssl_cipher_iv_length($this->cipher);

        $iv = substr($data, 0, $ivLength);
        $tag = substr($data, $ivLength, 16);
        $ciphertext = substr($data, $ivLength + 16);

        return openssl_decrypt(
            $ciphertext,
            $this->cipher,
            $this->key,
            OPENSSL_RAW_DATA,
            $iv,
            $tag
        );
    }

    public function hash(string $data): string
    {
        return password_hash($data, PASSWORD_ARGON2ID, [
            'memory_cost' => 65536,
            'time_cost' => 4,
            'threads' => 3
        ]);
    }

    public function verify(string $data, string $hash): bool
    {
        return password_verify($data, $hash);
    }

    public function generateApiKey(): string
    {
        return bin2hex(random_bytes(32));
    }

    public function hashPassword(string $password): string
    {
        return password_hash($password, PASSWORD_ARGON2ID);
    }

    private function getOrCreateKey(): string
    {
        $keyFile = dirname(__DIR__, 3) . '/storage/.encryption_key';

        if (file_exists($keyFile)) {
            return file_get_contents($keyFile);
        }

        $key = bin2hex(random_bytes(32));
        file_put_contents($keyFile, $key);
        chmod($keyFile, 0600);

        return $key;
    }
}
```

## Step 2: Secure Data Storage

```php
<?php
// src/Service/SecureDataService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class SecureDataService
{
    private $encryption;

    public function __construct()
    {
        $this->encryption = new EncryptionService();
    }

    public function storeSensitiveData(int $clientId, string $type, string $data): void
    {
        $encrypted = $this->encryption->encrypt($data);

        Capsule::table('mod_sensitive_data')->insert([
            'client_id' => $clientId,
            'data_type' => $type,
            'encrypted_data' => $encrypted,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    public function retrieveSensitiveData(int $clientId, string $type): ?string
    {
        $record = Capsule::table('mod_sensitive_data')
            ->where('client_id', $clientId)
            ->where('data_type', $type)
            ->first();

        if (!$record) {
            return null;
        }

        return $this->encryption->decrypt($record->encrypted_data);
    }

    public function deleteSensitiveData(int $clientId, string $type): void
    {
        Capsule::table('mod_sensitive_data')
            ->where('client_id', $clientId)
            ->where('data_type', $type)
            ->delete();
    }

    public function maskCreditCard(string $number): string
    {
        return '****' . substr($number, -4);
    }

    public function maskSsn(string $ssn): string
    {
        return '***-**-' . substr($ssn, -4);
    }
}
```

## Verification Checklist

- [ ] Encryption service implemented
- [ ] Encryption/decryption working
- [ ] Key management secure
- [ ] Sensitive data storage working
- [ ] Data masking implemented
- [ ] Test encryption successful
