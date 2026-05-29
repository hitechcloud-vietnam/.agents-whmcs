# WHMCS Video Conferencing DevKit

## Overview

Video conferencing integration for WHMCS enabling instant meeting creation, calendar integration, and meeting history tracking.

## Module Files

```php
<?php
/**
 * WHMCS Video Conferencing Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/VideoConferencing.php';

function whmcs_video_conferencing_activate() {
    $vc = new VideoConferencing();
    return $vc->activate();
}

function whmcs_video_create_meeting($userId, $title, $duration = 60) {
    $vc = new VideoConferencing();
    return $vc->createMeeting($userId, $title, $duration);
}
```

### lib/VideoConferencing.php

```php
<?php
namespace WHMCS\Module\VideoConferencing;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class VideoConferencing {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Video Conferencing module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_meetings` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `meeting_code` VARCHAR(50) NOT NULL,
                `title` VARCHAR(255) NOT NULL,
                `host_id` INT UNSIGNED NOT NULL,
                `provider` VARCHAR(50) NOT NULL DEFAULT 'zoom',
                `meeting_id` VARCHAR(100) NULL,
                `join_url` VARCHAR(500) NULL,
                `start_time` DATETIME NOT NULL,
                `duration_minutes` INT UNSIGNED NOT NULL DEFAULT 60,
                `status` ENUM('scheduled', 'in_progress', 'ended', 'cancelled') NOT NULL DEFAULT 'scheduled',
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_meeting_participants` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `meeting_id` INT UNSIGNED NOT NULL,
                `user_id` INT UNSIGNED NULL,
                `name` VARCHAR(255) NOT NULL,
                `email` VARCHAR(255) NOT NULL,
                `joined_at` DATETIME NULL,
                `left_at` DATETIME NULL,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function createMeeting($userId, $title, $duration = 60) {
        $id = Capsule::table('mod_meetings')->insertGetId([
            'meeting_code' => 'MTG-' . substr(uniqid(), -8),
            'title' => $title,
            'host_id' => $userId,
            'start_time' => Carbon::now(),
            'duration_minutes' => $duration,
        ]);
        
        return ['success' => true, 'meeting_id' => $id];
    }
}
```

## API Endpoints

```
POST /api/v1/meetings                    - Create meeting
GET  /api/v1/meetings                  - List meetings
POST /api/v1/meetings/{id}/join        - Join meeting
```
