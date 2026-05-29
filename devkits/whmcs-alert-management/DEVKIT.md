# WHMCS Alert Management DevKit

## Overview

Comprehensive alert management system for WHMCS enabling creation, routing, and resolution of operational alerts.

## Module Files

```php
<?php
/**
 * WHMCS Alert Management Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/AlertManager.php';

function whmcs_alert_management_activate() {
    $manager = new AlertManager();
    return $manager->activate();
}

function whmcs_alert_create($data) {
    $manager = new AlertManager();
    return $manager->createAlert($data);
}

function whmcs_alert_acknowledge($alertId) {
    $manager = new AlertManager();
    return $manager->acknowledgeAlert($alertId);
}
```

### lib/AlertManager.php

```php
<?php
namespace WHMCS\Module\AlertManagement;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class AlertManager {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Alert Management module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_alerts` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `alert_code` VARCHAR(50) NOT NULL,
                `alert_type` VARCHAR(100) NOT NULL,
                `severity` ENUM('info', 'warning', 'error', 'critical') NOT NULL DEFAULT 'warning',
                `title` VARCHAR(255) NOT NULL,
                `description` TEXT NULL,
                `source` VARCHAR(100) NULL,
                `status` ENUM('new', 'acknowledged', 'in_progress', 'resolved') NOT NULL DEFAULT 'new',
                `assigned_to` INT UNSIGNED NULL,
                `resolved_at` DATETIME NULL,
                `resolved_by` INT UNSIGNED NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_alert_rules` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `rule_name` VARCHAR(255) NOT NULL,
                `conditions` JSON NOT NULL,
                `severity` VARCHAR(20) NOT NULL DEFAULT 'warning',
                `notification_channels` JSON NULL,
                `auto_assign_to` INT UNSIGNED NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function createAlert($data) {
        $id = Capsule::table('mod_alerts')->insertGetId([
            'alert_code' => 'ALR-' . date('Ymd') . '-' . strtoupper(substr(uniqid(), -6)),
            'alert_type' => $data['type'],
            'severity' => $data['severity'] ?? 'warning',
            'title' => $data['title'],
            'description' => $data['description'] ?? null,
            'source' => $data['source'] ?? null,
        ]);
        
        return ['success' => true, 'alert_id' => $id];
    }
    
    public function acknowledgeAlert($alertId, $userId) {
        Capsule::table('mod_alerts')
            ->where('id', $alertId)
            ->update([
                'status' => 'acknowledged',
                'assigned_to' => $userId,
            ]);
        
        return ['success' => true];
    }
    
    public function resolveAlert($alertId, $userId) {
        Capsule::table('mod_alerts')
            ->where('id', $alertId)
            ->update([
                'status' => 'resolved',
                'resolved_at' => Carbon::now(),
                'resolved_by' => $userId,
            ]);
        
        return ['success' => true];
    }
    
    public function getActiveAlerts($severity = null) {
        $query = Capsule::table('mod_alerts')
            ->whereIn('status', ['new', 'acknowledged', 'in_progress']);
        
        if ($severity) {
            $query->where('severity', $severity);
        }
        
        return $query->orderBy('severity', 'desc')
            ->orderBy('created_at', 'desc')
            ->get();
    }
}
```

## API Endpoints

```
POST /api/v1/alerts                       - Create alert
GET  /api/v1/alerts                     - List alerts
POST /api/v1/alerts/{id}/acknowledge      - Acknowledge
POST /api/v1/alerts/{id}/resolve         - Resolve
```
