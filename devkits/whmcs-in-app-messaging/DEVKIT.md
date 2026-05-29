# WHMCS In-App Messaging DevKit

## Overview

In-app messaging system for WHMCS enabling real-time messaging, chat functionality, announcements, and user-to-user communication.

## Features

- Real-time messaging
- Chat rooms
- Direct messaging
- Announcements
- Message templates
- Read receipts
- Typing indicators
- Attachments
- Message search
- Notification integration

## Module Files

```php
<?php
/**
 * WHMCS In-App Messaging Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/InAppMessaging.php';

function whmcs_in_app_messaging_activate() {
    $messaging = new InAppMessaging();
    return $messaging->activate();
}

function whmcs_in_app_messaging_send($senderId, $recipientId, $message) {
    $messaging = new InAppMessaging();
    return $messaging->sendMessage($senderId, $recipientId, $message);
}

function whmcs_in_app_messaging_get_conversations($userId) {
    $messaging = new InAppMessaging();
    return $messaging->getConversations($userId);
}
```

### lib/InAppMessaging.php

```php
<?php
namespace WHMCS\Module\InAppMessaging;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class InAppMessaging {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'In-App Messaging module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_messaging_conversations` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `conversation_type` ENUM('direct', 'support', 'announcement') NOT NULL DEFAULT 'direct',
                `subject` VARCHAR(255) NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                `last_message_at` DATETIME NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_messaging_participants` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `conversation_id` INT UNSIGNED NOT NULL,
                `user_id` INT UNSIGNED NOT NULL,
                `last_read_at` DATETIME NULL,
                `notifications_enabled` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_conv_user` (`conversation_id`, `user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_messaging_messages` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `conversation_id` INT UNSIGNED NOT NULL,
                `sender_id` INT UNSIGNED NOT NULL,
                `message` TEXT NOT NULL,
                `attachment_url` VARCHAR(500) NULL,
                `is_edited` TINYINT(1) NOT NULL DEFAULT 0,
                `is_deleted` TINYINT(1) NOT NULL DEFAULT 0,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                INDEX `idx_conversation` (`conversation_id`, `created_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function sendMessage($senderId, $recipientId, $message) {
        $conversation = $this->getOrCreateDirectConversation($senderId, $recipientId);
        
        $messageId = Capsule::table('mod_messaging_messages')->insertGetId([
            'conversation_id' => $conversation->id,
            'sender_id' => $senderId,
            'message' => $message,
        ]);
        
        Capsule::table('mod_messaging_conversations')
            ->where('id', $conversation->id)
            ->update(['last_message_at' => Carbon::now()]);
        
        return ['success' => true, 'message_id' => $messageId];
    }
    
    protected function getOrCreateDirectConversation($user1, $user2) {
        $existing = Capsule::table('mod_messaging_participants as p1')
            ->join('mod_messaging_participants as p2', 'p1.conversation_id', '=', 'p2.conversation_id')
            ->where('p1.user_id', $user1)
            ->where('p2.user_id', $user2)
            ->where('p1.conversation_id', '=', 'p2.conversation_id')
            ->first();
        
        if ($existing) {
            return Capsule::table('mod_messaging_conversations')->where('id', $existing->conversation_id)->first();
        }
        
        $convId = Capsule::table('mod_messaging_conversations')->insertGetId([
            'conversation_type' => 'direct',
        ]);
        
        Capsule::table('mod_messaging_participants')->insert([
            ['conversation_id' => $convId, 'user_id' => $user1],
            ['conversation_id' => $convId, 'user_id' => $user2],
        ]);
        
        return Capsule::table('mod_messaging_conversations')->where('id', $convId)->first();
    }
    
    public function getConversations($userId) {
        return Capsule::table('mod_messaging_participants as p')
            ->join('mod_messaging_conversations as c', 'p.conversation_id', '=', 'c.id')
            ->where('p.user_id', $userId)
            ->where('c.is_active', 1)
            ->select('c.*')
            ->orderBy('c.last_message_at', 'desc')
            ->get();
    }
    
    public function getMessages($conversationId, $userId, $limit = 50) {
        // Verify user is participant
        $participant = Capsule::table('mod_messaging_participants')
            ->where('conversation_id', $conversationId)
            ->where('user_id', $userId)
            ->exists();
        
        if (!$participant) {
            return ['error' => 'Access denied'];
        }
        
        return Capsule::table('mod_messaging_messages')
            ->where('conversation_id', $conversationId)
            ->where('is_deleted', 0)
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get();
    }
}
```

## API Endpoints

```
POST /api/v1/messaging/send              - Send message
GET  /api/v1/messaging/conversations    - Get conversations
GET  /api/v1/messaging/conversations/{id}/messages - Get messages
POST /api/v1/messaging/conversations    - Create conversation
```
