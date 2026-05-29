# WHMCS Webinar Integration DevKit

## Overview

Webinar integration system for WHMCS enabling scheduling, registration, and tracking of webinar attendance for marketing and training purposes.

## Features

- Webinar scheduling
- Registration management
- Attendance tracking
- Email reminders
- Follow-up automation
- Integration with Zoom/Teams

## Module Files

```php
<?php
/**
 * WHMCS Webinar Integration Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/WebinarIntegration.php';

function whmcs_webinar_integration_activate() {
    $integration = new WebinarIntegration();
    return $integration->activate();
}

function whmcs_webinar_create($data) {
    $integration = new WebinarIntegration();
    return $integration->createWebinar($data);
}
```

### lib/WebinarIntegration.php

```php
<?php
namespace WHMCS\Module\WebinarIntegration;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class WebinarIntegration {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Webinar Integration module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_webinars` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `webinar_name` VARCHAR(255) NOT NULL,
                `webinar_code` VARCHAR(50) NOT NULL,
                `description` TEXT NULL,
                `provider` ENUM('zoom', 'teams', 'webex', 'custom') NOT NULL DEFAULT 'zoom',
                `meeting_id` VARCHAR(100) NULL,
                `join_url` VARCHAR(500) NULL,
                `start_time` DATETIME NOT NULL,
                `duration_minutes` INT UNSIGNED NOT NULL DEFAULT 60,
                `max_attendees` INT UNSIGNED DEFAULT 100,
                `status` ENUM('scheduled', 'live', 'ended', 'cancelled') NOT NULL DEFAULT 'scheduled',
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_webinar_registrations` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `webinar_id` INT UNSIGNED NOT NULL,
                `user_id` INT UNSIGNED NULL,
                `name` VARCHAR(255) NOT NULL,
                `email` VARCHAR(255) NOT NULL,
                `status` ENUM('registered', 'attended', 'no_show', 'cancelled') NOT NULL DEFAULT 'registered',
                `registered_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function createWebinar($data) {
        $id = Capsule::table('mod_webinars')->insertGetId([
            'webinar_name' => $data['name'],
            'webinar_code' => 'WEB-' . strtoupper(substr(uniqid(), -6)),
            'description' => $data['description'] ?? null,
            'provider' => $data['provider'] ?? 'zoom',
            'start_time' => $data['start_time'],
            'duration_minutes' => $data['duration'] ?? 60,
            'max_attendees' => $data['max_attendees'] ?? 100,
        ]);
        
        return ['success' => true, 'webinar_id' => $id];
    }
    
    public function registerUser($webinarId, $userId) {
        $client = Capsule::table('tblclients')->where('id', $userId)->first();
        
        $id = Capsule::table('mod_webinar_registrations')->insertGetId([
            'webinar_id' => $webinarId,
            'user_id' => $userId,
            'name' => $client->firstname . ' ' . $client->lastname,
            'email' => $client->email,
        ]);
        
        return ['success' => true, 'registration_id' => $id];
    }
}
```

## API Endpoints

```
POST /api/v1/webinars                    - Create webinar
GET  /api/v1/webinars                  - List webinars
POST /api/v1/webinars/{id}/register    - Register
```
