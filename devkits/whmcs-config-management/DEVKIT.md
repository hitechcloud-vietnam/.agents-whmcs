# WHMCS Config Management DevKit

## Overview

Configuration management system for WHMCS enabling centralized config storage, environment-based configs, config versioning, and secret management.

## Features

- Centralized config
- Environment configs
- Config versioning
- Secret rotation
- Config validation
- Import/Export
- Feature flags
- Dynamic config

## Module Files

```php
<?php
/**
 * WHMCS Config Management Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/ConfigManager.php';

function whmcs_config_management_activate() {
    $manager = new ConfigManager();
    return $manager->activate();
}

function whmcs_config_get($key, $default = null) {
    $manager = new ConfigManager();
    return $manager->get($key, $default);
}

function whmcs_config_set($key, $value, $env = null) {
    $manager = new ConfigManager();
    return $manager->set($key, $value, $env);
}
```

### lib/ConfigManager.php

```php
<?php
namespace WHMCS\Module\ConfigManagement;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class ConfigManager {
    
    protected $environment = 'production';
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Config Management module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_config_entries` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `config_key` VARCHAR(255) NOT NULL,
                `config_value` TEXT NULL,
                `environment` VARCHAR(50) NOT NULL DEFAULT 'default',
                `is_secret` TINYINT(1) NOT NULL DEFAULT 0,
                `is_encrypted` TINYINT(1) NOT NULL DEFAULT 0,
                `version` INT UNSIGNED NOT NULL DEFAULT 1,
                `description` VARCHAR(255) NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_key_env` (`config_key`, `environment`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_config_versions` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `config_id` INT UNSIGNED NOT NULL,
                `config_value` TEXT NULL,
                `version` INT UNSIGNED NOT NULL,
                `changed_by` INT UNSIGNED NULL,
                `change_reason` VARCHAR(255) NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function get($key, $default = null) {
        $entry = Capsule::table('mod_config_entries')
            ->where('config_key', $key)
            ->where(function($q) {
                $q->where('environment', $this->environment)
                    ->orWhere('environment', 'default');
            })
            ->orderBy('environment', 'desc')
            ->first();
        
        if (!$entry) {
            return $default;
        }
        
        $value = $entry->config_value;
        
        if ($entry->is_encrypted) {
            $value = $this->decrypt($value);
        }
        
        return $value;
    }
    
    public function set($key, $value, $env = null, $options = []) {
        $environment = $env ?? $this->environment;
        $isSecret = $options['secret'] ?? false;
        $isEncrypted = $options['encrypt'] ?? $isSecret;
        
        if ($isEncrypted) {
            $value = $this->encrypt($value);
        }
        
        $existing = Capsule::table('mod_config_entries')
            ->where('config_key', $key)
            ->where('environment', $environment)
            ->first();
        
        if ($existing) {
            // Create version before updating
            $this->createVersion($existing->id, $existing->config_value);
            
            Capsule::table('mod_config_entries')
                ->where('id', $existing->id)
                ->update([
                    'config_value' => $value,
                    'is_secret' => $isSecret,
                    'is_encrypted' => $isEncrypted,
                    'version' => Capsule::raw('version + 1'),
                ]);
        } else {
            Capsule::table('mod_config_entries')->insert([
                'config_key' => $key,
                'config_value' => $value,
                'environment' => $environment,
                'is_secret' => $isSecret,
                'is_encrypted' => $isEncrypted,
                'version' => 1,
            ]);
        }
        
        return ['success' => true];
    }
    
    protected function createVersion($configId, $value) {
        $config = Capsule::table('mod_config_entries')->where('id', $configId)->first();
        
        Capsule::table('mod_config_versions')->insert([
            'config_id' => $configId,
            'config_value' => $value,
            'version' => $config->version,
        ]);
    }
    
    protected function encrypt($value) {
        $key = defined('ENCRYPTION_KEY') ? ENCRYPTION_KEY : 'default-key';
        return openssl_encrypt($value, 'AES-256-CBC', $key, 0, substr(md5($key), 0, 16));
    }
    
    protected function decrypt($value) {
        $key = defined('ENCRYPTION_KEY') ? ENCRYPTION_KEY : 'default-key';
        return openssl_decrypt($value, 'AES-256-CBC', $key, 0, substr(md5($key), 0, 16));
    }
    
    public function getHistory($key) {
        $entry = Capsule::table('mod_config_entries')
            ->where('config_key', $key)
            ->first();
        
        if (!$entry) {
            return [];
        }
        
        return Capsule::table('mod_config_versions')
            ->where('config_id', $entry->id)
            ->orderBy('version', 'desc')
            ->get();
    }
}
```

## API Endpoints

```
GET  /api/v1/config/{key}               - Get config value
POST /api/v1/config                     - Set config value
GET  /api/v1/config/{key}/history       - Get config history
POST /api/v1/config/import              - Import configs
GET  /api/v1/config/export              - Export configs
```
