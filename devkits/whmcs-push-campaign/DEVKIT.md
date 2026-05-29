# WHMCS Push Campaign DevKit

## Overview

Push notification campaign management system for WHMCS enabling targeted push notifications, automated campaigns, and engagement tracking.

## Features

- Push notification campaigns
- Device registration
- Segmentation
- Scheduled delivery
- Rich notifications
- Click tracking
- Conversion tracking
- A/B testing
- Opt-in management

## Module Files

```php
<?php
/**
 * WHMCS Push Campaign Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/PushCampaignManager.php';

function whmcs_push_campaign_activate() {
    $manager = new PushCampaignManager();
    return $manager->activate();
}

function whmcs_push_campaign_send($userId, $title, $body, $options = []) {
    $manager = new PushCampaignManager();
    return $manager->sendPush($userId, $title, $body, $options);
}

function whmcs_push_campaign_create($data) {
    $manager = new PushCampaignManager();
    return $manager->createCampaign($data);
}
```

### lib/PushCampaignManager.php

```php
<?php
namespace WHMCS\Module\PushCampaign;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class PushCampaignManager {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Push Campaign module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_push_campaigns` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `campaign_name` VARCHAR(255) NOT NULL,
                `title` VARCHAR(255) NOT NULL,
                `body` TEXT NOT NULL,
                `icon` VARCHAR(500) NULL,
                `image` VARCHAR(500) NULL,
                `action_url` VARCHAR(500) NULL,
                `target_segment` JSON NULL,
                `status` VARCHAR(20) NOT NULL DEFAULT 'draft',
                `scheduled_at` DATETIME NULL,
                `sent_count` INT UNSIGNED DEFAULT 0,
                `delivered_count` INT UNSIGNED DEFAULT 0,
                `clicked_count` INT UNSIGNED DEFAULT 0,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_push_devices` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `user_id` INT UNSIGNED NOT NULL,
                `device_token` VARCHAR(500) NOT NULL,
                `device_type` ENUM('ios', 'android', 'web') NOT NULL,
                `subscription_status` ENUM('subscribed', 'unsubscribed') NOT NULL DEFAULT 'subscribed',
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_device_token` (`device_token`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_push_logs` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `campaign_id` INT UNSIGNED NULL,
                `device_id` INT UNSIGNED NOT NULL,
                `status` VARCHAR(20) NOT NULL DEFAULT 'pending',
                `sent_at` DATETIME NULL,
                `delivered_at` DATETIME NULL,
                `clicked_at` DATETIME NULL,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function sendPush($userId, $title, $body, $options = []) {
        $devices = Capsule::table('mod_push_devices')
            ->where('user_id', $userId)
            ->where('subscription_status', 'subscribed')
            ->get();
        
        foreach ($devices as $device) {
            $this->sendToDevice($device, $title, $body, $options);
        }
        
        return ['success' => true, 'devices' => count($devices)];
    }
    
    protected function sendToDevice($device, $title, $body, $options) {
        // Integrate with FCM, APNS, etc.
        $logId = Capsule::table('mod_push_logs')->insertGetId([
            'device_id' => $device->id,
            'status' => 'sent',
            'sent_at' => Carbon::now(),
        ]);
        
        return $logId;
    }
    
    public function createCampaign($data) {
        $id = Capsule::table('mod_push_campaigns')->insertGetId([
            'campaign_name' => $data['name'],
            'title' => $data['title'],
            'body' => $data['body'],
            'action_url' => $data['action_url'] ?? null,
            'status' => 'draft',
        ]);
        
        return ['success' => true, 'campaign_id' => $id];
    }
}
```

## API Endpoints

```
POST /api/v1/push/send                  - Send push notification
POST /api/v1/push/campaign              - Create campaign
POST /api/v1/push/campaign/{id}/send     - Send campaign
GET  /api/v1/push/devices              - List registered devices
POST /api/v1/push/register             - Register device
POST /api/v1/push/unregister           - Unregister device
```
