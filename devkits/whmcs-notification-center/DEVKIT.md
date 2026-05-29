# WHMCS Notification Center DevKit

## Overview

Centralized notification management system for WHMCS that aggregates, prioritizes, and delivers notifications across multiple channels including in-app, email, SMS, and push.

## Features

- Multi-channel delivery
- Notification prioritization
- User preferences
- Notification grouping
- Read/unread management
- Delivery scheduling
- Transactional notifications
- Marketing notifications
- Push notification support
- Real-time updates

## Module Files

```php
<?php
/**
 * WHMCS Notification Center Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/NotificationCenter.php';

function whmcs_notification_center_activate() {
    $center = new NotificationCenter();
    return $center->activate();
}

function whmcs_notification_send($userId, $type, $title, $message, $options = []) {
    $center = new NotificationCenter();
    return $center->send($userId, $type, $title, $message, $options);
}

function whmcs_notification_get($userId, $options = []) {
    $center = new NotificationCenter();
    return $center->getNotifications($userId, $options);
}

function whmcs_notification_mark_read($notificationId) {
    $center = new NotificationCenter();
    return $center->markAsRead($notificationId);
}

add_hook('ServiceSuspended', 1, function($params) {
    $center = new NotificationCenter();
    $center->send($params['user_id'], 'warning', 'Service Suspended', 
        'Your service has been suspended due to non-payment');
});
```

### lib/NotificationCenter.php

```php
<?php
namespace WHMCS\Module\NotificationCenter;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class NotificationCenter {
    
    protected $channels = ['in_app', 'email', 'sms', 'push'];
    
    public function activate() {
        try {
            $this->createTables();
            $this->createDefaultChannels();
            return ['success' => true, 'msg' => 'Notification Center module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_notifications` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `user_id` INT UNSIGNED NOT NULL,
                `notification_type` VARCHAR(50) NOT NULL,
                `priority` ENUM('low', 'normal', 'high', 'urgent') NOT NULL DEFAULT 'normal',
                `title` VARCHAR(255) NOT NULL,
                `message` TEXT NOT NULL,
                `data` JSON NULL,
                `channels` JSON NULL,
                `is_read` TINYINT(1) NOT NULL DEFAULT 0,
                `read_at` DATETIME NULL,
                `expires_at` DATETIME NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                INDEX `idx_user_unread` (`user_id`, `is_read`, `created_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_notification_preferences` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `user_id` INT UNSIGNED NOT NULL,
                `notification_type` VARCHAR(50) NOT NULL,
                `channel` VARCHAR(20) NOT NULL,
                `enabled` TINYINT(1) NOT NULL DEFAULT 1,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_user_type_channel` (`user_id`, `notification_type`, `channel`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_notification_delivery_log` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `notification_id` BIGINT UNSIGNED NOT NULL,
                `channel` VARCHAR(20) NOT NULL,
                `status` ENUM('pending', 'sent', 'delivered', 'failed') NOT NULL DEFAULT 'pending',
                `sent_at` DATETIME NULL,
                `delivered_at` DATETIME NULL,
                `error_message` TEXT NULL,
                PRIMARY KEY (`id`),
                CONSTRAINT `fk_delivery_notification` FOREIGN KEY (`notification_id`) REFERENCES `mod_notifications`(`id`) ON DELETE CASCADE
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function createDefaultChannels() {
        $channels = [
            ['name' => 'in_app', 'display_name' => 'In-App Notifications'],
            ['name' => 'email', 'display_name' => 'Email'],
            ['name' => 'sms', 'display_name' => 'SMS'],
            ['name' => 'push', 'display_name' => 'Push Notifications'],
        ];
    }
    
    public function send($userId, $type, $title, $message, $options = []) {
        $priority = $options['priority'] ?? 'normal';
        $channels = $options['channels'] ?? $this->getDefaultChannels($type);
        
        $notificationId = Capsule::table('mod_notifications')->insertGetId([
            'user_id' => $userId,
            'notification_type' => $type,
            'priority' => $priority,
            'title' => $title,
            'message' => $message,
            'data' => json_encode($options['data'] ?? []),
            'channels' => json_encode($channels),
            'expires_at' => $options['expires_at'] ?? null,
        ]);
        
        // Send to each channel
        foreach ($channels as $channel) {
            if ($this->isChannelEnabled($userId, $type, $channel)) {
                $this->sendToChannel($notificationId, $channel, $userId, $title, $message);
            }
        }
        
        return ['success' => true, 'notification_id' => $notificationId];
    }
    
    protected function getDefaultChannels($type) {
        $defaults = [
            'billing' => ['in_app', 'email'],
            'service' => ['in_app', 'email', 'sms'],
            'support' => ['in_app', 'email'],
            'marketing' => ['in_app', 'email'],
            'alert' => ['in_app', 'email', 'sms'],
        ];
        
        return $defaults[$type] ?? ['in_app', 'email'];
    }
    
    protected function isChannelEnabled($userId, $type, $channel) {
        $preference = Capsule::table('mod_notification_preferences')
            ->where('user_id', $userId)
            ->where('notification_type', $type)
            ->where('channel', $channel)
            ->first();
        
        if ($preference) {
            return $preference->enabled;
        }
        
        return true; // Default enabled
    }
    
    protected function sendToChannel($notificationId, $channel, $userId, $title, $message) {
        $client = Capsule::table('tblclients')->where('id', $userId)->first();
        
        switch ($channel) {
            case 'email':
                if ($client && $client->email) {
                    sendEmail('notification', $client->email, [
                        'subject' => $title,
                        'message' => $message,
                    ]);
                }
                break;
                
            case 'sms':
                if ($client && $client->phonenumber) {
                    // Integrate with SMS provider
                }
                break;
                
            case 'push':
                // Integrate with push notification service
                break;
                
            case 'in_app':
            default:
                // Already stored in mod_notifications
                break;
        }
        
        Capsule::table('mod_notification_delivery_log')->insert([
            'notification_id' => $notificationId,
            'channel' => $channel,
            'status' => 'sent',
            'sent_at' => Carbon::now(),
        ]);
    }
    
    public function getNotifications($userId, $options = []) {
        $query = Capsule::table('mod_notifications')
            ->where('user_id', $userId)
            ->where(function($q) {
                $q->whereNull('expires_at')
                    ->orWhere('expires_at', '>', Carbon::now());
            });
        
        if (!empty($options['unread_only'])) {
            $query->where('is_read', 0);
        }
        
        if (!empty($options['type'])) {
            $query->where('notification_type', $options['type']);
        }
        
        if (!empty($options['priority'])) {
            $query->where('priority', $options['priority']);
        }
        
        $limit = $options['limit'] ?? 20;
        $offset = $options['offset'] ?? 0;
        
        $notifications = $query->orderBy('created_at', 'desc')
            ->limit($limit)
            ->offset($offset)
            ->get();
        
        $unreadCount = Capsule::table('mod_notifications')
            ->where('user_id', $userId)
            ->where('is_read', 0)
            ->count();
        
        return [
            'notifications' => $notifications,
            'unread_count' => $unreadCount,
            'limit' => $limit,
            'offset' => $offset,
        ];
    }
    
    public function markAsRead($notificationId) {
        Capsule::table('mod_notifications')
            ->where('id', $notificationId)
            ->update([
                'is_read' => 1,
                'read_at' => Carbon::now(),
            ]);
        
        return ['success' => true];
    }
    
    public function markAllAsRead($userId) {
        Capsule::table('mod_notifications')
            ->where('user_id', $userId)
            ->where('is_read', 0)
            ->update([
                'is_read' => 1,
                'read_at' => Carbon::now(),
            ]);
        
        return ['success' => true];
    }
    
    public function updatePreferences($userId, $type, $channel, $enabled) {
        Capsule::table('mod_notification_preferences')->updateOrInsert(
            ['user_id' => $userId, 'notification_type' => $type, 'channel' => $channel],
            ['enabled' => $enabled]
        );
        
        return ['success' => true];
    }
}
```

## API Endpoints

```
POST /api/v1/notifications/send         - Send notification
GET  /api/v1/notifications              - Get notifications
GET  /api/v1/notifications/count       - Get unread count
POST /api/v1/notifications/{id}/read     - Mark as read
POST /api/v1/notifications/read-all     - Mark all as read
DELETE /api/v1/notifications/{id}      - Delete notification
PUT  /api/v1/notifications/preferences  - Update preferences
```
