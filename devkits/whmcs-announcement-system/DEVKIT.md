# WHMCS Announcement System DevKit

## Overview

Announcement broadcasting system for WHMCS enabling administrators to create, schedule, and deliver announcements to users or specific segments.

## Features

- Rich announcements
- Scheduling
- Targeting
- Priority levels
- Dismissable banners
- Email integration
- Views tracking
- Category management

## Module Files

```php
<?php
/**
 * WHMCS Announcement System Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/AnnouncementManager.php';

function whmcs_announcement_system_activate() {
    $manager = new AnnouncementManager();
    return $manager->activate();
}

function whmcs_announcement_create($data) {
    $manager = new AnnouncementManager();
    return $manager->createAnnouncement($data);
}

function whmcs_announcement_get_active($userId = null) {
    $manager = new AnnouncementManager();
    return $manager->getActiveAnnouncements($userId);
}
```

### lib/AnnouncementManager.php

```php
<?php
namespace WHMCS\Module\AnnouncementSystem;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class AnnouncementManager {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Announcement System module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_announcements` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `title` VARCHAR(255) NOT NULL,
                `content` TEXT NOT NULL,
                `announcement_type` ENUM('info', 'warning', 'success', 'error', 'maintenance') NOT NULL DEFAULT 'info',
                `priority` ENUM('low', 'normal', 'high', 'urgent') NOT NULL DEFAULT 'normal',
                `target_audience` ENUM('all', 'logged_in', 'specific_users', 'specific_plans') NOT NULL DEFAULT 'all',
                `target_users` JSON NULL,
                `target_plans` JSON NULL,
                `start_date` DATETIME NOT NULL,
                `end_date` DATETIME NULL,
                `is_pinned` TINYINT(1) NOT NULL DEFAULT 0,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                `created_by` INT UNSIGNED NOT NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_announcement_views` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `announcement_id` INT UNSIGNED NOT NULL,
                `user_id` INT UNSIGNED NOT NULL,
                `viewed_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_announcement_user` (`announcement_id`, `user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_announcement_dismissals` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `announcement_id` INT UNSIGNED NOT NULL,
                `user_id` INT UNSIGNED NOT NULL,
                `dismissed_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_announcement_user` (`announcement_id`, `user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function createAnnouncement($data) {
        $id = Capsule::table('mod_announcements')->insertGetId([
            'title' => $data['title'],
            'content' => $data['content'],
            'announcement_type' => $data['type'] ?? 'info',
            'priority' => $data['priority'] ?? 'normal',
            'target_audience' => $data['audience'] ?? 'all',
            'target_users' => json_encode($data['target_users'] ?? []),
            'target_plans' => json_encode($data['target_plans'] ?? []),
            'start_date' => $data['start_date'] ?? Carbon::now(),
            'end_date' => $data['end_date'] ?? null,
            'is_pinned' => $data['pinned'] ?? 0,
            'created_by' => $data['admin_id'] ?? 0,
        ]);
        
        return ['success' => true, 'announcement_id' => $id];
    }
    
    public function getActiveAnnouncements($userId = null) {
        $now = Carbon::now();
        
        $query = Capsule::table('mod_announcements')
            ->where('is_active', 1)
            ->where('start_date', '<=', $now)
            ->where(function($q) use ($now) {
                $q->whereNull('end_date')
                    ->orWhere('end_date', '>=', $now);
            })
            ->orderBy('is_pinned', 'desc')
            ->orderBy('priority', 'desc')
            ->orderBy('created_at', 'desc');
        
        if ($userId) {
            $query->where(function($q) use ($userId) {
                $q->where('target_audience', 'all')
                    ->orWhere(function($sq) use ($userId) {
                        $sq->where('target_audience', 'logged_in');
                    });
            });
            
            // Exclude dismissed
            $dismissed = Capsule::table('mod_announcement_dismissals')
                ->where('user_id', $userId)
                ->pluck('announcement_id')
                ->toArray();
            
            if (!empty($dismissed)) {
                $query->whereNotIn('id', $dismissed);
            }
        }
        
        return $query->get();
    }
    
    public function dismiss($announcementId, $userId) {
        Capsule::table('mod_announcement_dismissals')->insert([
            'announcement_id' => $announcementId,
            'user_id' => $userId,
        ]);
        
        return ['success' => true];
    }
    
    public function trackView($announcementId, $userId) {
        Capsule::table('mod_announcement_views')->insert([
            'announcement_id' => $announcementId,
            'user_id' => $userId,
        ]);
        
        return ['success' => true];
    }
}
```

## API Endpoints

```
POST /api/v1/announcements               - Create announcement
GET  /api/v1/announcements/active        - Get active announcements
POST /api/v1/announcements/{id}/dismiss  - Dismiss announcement
POST /api/v1/announcements/{id}/view      - Track view
```
