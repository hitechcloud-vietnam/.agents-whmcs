# WHMCS Backup Encryption Workflow

## Overview
This workflow implements encrypted backups for WHMCS.

## Prerequisites
- WHMCS with backup capabilities
- Encryption keys available
- Secure storage

## Step-by-Step Process

### Step 1: Create Encrypted Backup
```php
<?php
// /includes/backup/EncryptedBackupManager.php

class EncryptedBackupManager {
    private $encryptionKey;

    public function __construct()
    {
        $this->encryptionKey = getConfig('backup_encryption_key');
    }

    /**
     * Create encrypted backup
     */
    public function createEncryptedBackup(string $sourcePath): string
    {
        $tempBackup = $this->createBackup($sourcePath);
        $encryptedPath = $this->encryptFile($tempBackup);
        unlink($tempBackup);

        return $encryptedPath;
    }

    private function encryptFile(string $path): string
    {
        $encryptedPath = $path . '.enc';
        $key = base64_decode($this->encryptionKey);
        $iv = random_bytes(16);

        $content = file_get_contents($path);
        $encrypted = openssl_encrypt($content, 'AES-256-CBC', $key, OPENSSL_RAW_DATA, $iv);

        file_put_contents($encryptedPath, $iv . $encrypted);

        return $encryptedPath;
    }
}
```

## Related Workflows
- [WHMCS Backup Automation](./whmcs-backup-automation.md)
- [WHMCS Key Management](./whmcs-key-management.md)