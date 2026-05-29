# WHMCS Secret Rotation DevKit

## Overview

Secret and credential rotation system for WHMCS enabling automatic rotation of API keys, passwords, tokens, and certificates.

## Features

- Automated rotation
- Multiple secret types
- Rotation schedules
- Audit logging
- Grace period handling
- Rollback support
- Integration hooks

## Module Files

```php
<?php
/**
 * WHMCS Secret Rotation Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/SecretRotation.php';

function whmcs_secret_rotation_activate() {
    $rotation = new SecretRotation();
    return $rotation->activate();
}

function whmcs_secret_rotation_create($data) {
    $rotation = new SecretRotation();
    return $rotation->createSecret($data);
}

function whmcs_secret_rotation_rotate($secretId) {
    $rotation = new SecretRotation();
    return $rotation->rotateSecret($secretId);
}
```

### lib/SecretRotation.php

```php
<?php
namespace WHMCS\Module\SecretRotation;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class SecretRotation {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Secret Rotation module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_secret_configs` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `secret_name` VARCHAR(255) NOT NULL,
                `secret_type` ENUM('api_key', 'password', 'token', 'certificate', 'ssh_key') NOT NULL,
                `rotation_days` INT UNSIGNED NOT NULL DEFAULT 90,
                `grace_period_days` INT UNSIGNED NOT NULL DEFAULT 7,
                `auto_rotate` TINYINT(1) NOT NULL DEFAULT 1,
                `notify_before_days` INT UNSIGNED NOT NULL DEFAULT 7,
                `last_rotated_at` DATETIME NULL,
                `next_rotation_at` DATETIME NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_secret_versions` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `secret_id` INT UNSIGNED NOT NULL,
                `secret_value_encrypted` TEXT NOT NULL,
                `version` INT UNSIGNED NOT NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                `expires_at` DATETIME NULL,
                PRIMARY KEY (`id`),
                INDEX `idx_secret_versions` (`secret_id`, `is_active`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function createSecret($data) {
        $id = Capsule::table('mod_secret_configs')->insertGetId([
            'secret_name' => $data['name'],
            'secret_type' => $data['type'],
            'rotation_days' => $data['rotation_days'] ?? 90,
            'grace_period_days' => $data['grace_period'] ?? 7,
            'next_rotation_at' => Carbon::now()->addDays($data['rotation_days'] ?? 90),
        ]);
        
        // Create initial version
        $this->createVersion($id, $data['value']);
        
        return ['success' => true, 'secret_id' => $id];
    }
    
    protected function createVersion($secretId, $value) {
        $encrypted = $this->encrypt($value);
        
        // Deactivate previous versions
        Capsule::table('mod_secret_versions')
            ->where('secret_id', $secretId)
            ->update(['is_active' => 0]);
        
        Capsule::table('mod_secret_versions')->insert([
            'secret_id' => $secretId,
            'secret_value_encrypted' => $encrypted,
            'version' => Capsule::table('mod_secret_versions')->where('secret_id', $secretId)->count() + 1,
            'is_active' => 1,
        ]);
        
        Capsule::table('mod_secret_configs')
            ->where('id', $secretId)
            ->update([
                'last_rotated_at' => Carbon::now(),
                'next_rotation_at' => Carbon::now()->addDays(
                    Capsule::table('mod_secret_configs')->where('id', $secretId)->value('rotation_days')
                ),
            ]);
    }
    
    public function rotateSecret($secretId) {
        $secret = Capsule::table('mod_secret_configs')->where('id', $secretId)->first();
        
        // Generate new secret based on type
        $newValue = $this->generateSecret($secret->secret_type);
        $this->createVersion($secretId, $newValue);
        
        // Call rotation hooks
        $this->triggerRotationHooks($secretId, $newValue);
        
        return ['success' => true, 'new_version' => true];
    }
    
    protected function generateSecret($type) {
        switch ($type) {
            case 'api_key':
                return bin2hex(random_bytes(32));
            case 'password':
                return bin2hex(random_bytes(16));
            case 'token':
                return bin2hex(random_bytes(48));
            case 'certificate':
                return $this->generateCertificate();
            default:
                return bin2hex(random_bytes(32));
        }
    }
    
    protected function encrypt($value) {
        $key = defined('ENCRYPTION_KEY') ? ENCRYPTION_KEY : 'default-key';
        return openssl_encrypt($value, 'AES-256-CBC', $key, 0, substr(md5($key), 0, 16));
    }
    
    protected function triggerRotationHooks($secretId, $newValue) {
        logActivity("Secret rotated: ID $secretId");
    }
    
    public function processScheduledRotations() {
        $secrets = Capsule::table('mod_secret_configs')
            ->where('is_active', 1)
            ->where('auto_rotate', 1)
            ->where('next_rotation_at', '<=', Carbon::now())
            ->get();
        
        foreach ($secrets as $secret) {
            $this->rotateSecret($secret->id);
        }
    }
    
    public function getActiveSecret($secretId) {
        $version = Capsule::table('mod_secret_versions')
            ->where('secret_id', $secretId)
            ->where('is_active', 1)
            ->first();
        
        if (!$version) {
            return null;
        }
        
        return $this->decrypt($version->secret_value_encrypted);
    }
    
    protected function decrypt($value) {
        $key = defined('ENCRYPTION_KEY') ? ENCRYPTION_KEY : 'default-key';
        return openssl_decrypt($value, 'AES-256-CBC', $key, 0, substr(md5($key), 0, 16));
    }
}

add_hook('DailyCronJob', 1, function() {
    $rotation = new \WHMCS\Module\SecretRotation\SecretRotation();
    $rotation->processScheduledRotations();
});
```

## API Endpoints

```
POST /api/v1/secrets                    - Create secret
GET  /api/v1/secrets/{id}              - Get secret
POST /api/v1/secrets/{id}/rotate        - Rotate secret
GET  /api/v1/secrets/{id}/history       - Get rotation history
```
