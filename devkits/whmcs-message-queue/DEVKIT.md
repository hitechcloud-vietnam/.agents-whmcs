# WHMCS Message Queue DevKit

## Overview

Message queue system for WHMCS enabling asynchronous processing, task scheduling, and event-driven architecture.

## Features

- Async task processing
- Retry logic
- Priority queues
- Dead letter handling
- Task scheduling
- Batch processing
- Webhook delivery
- Event subscriptions

## Module Files

```php
<?php
/**
 * WHMCS Message Queue Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/MessageQueue.php';

function whmcs_message_queue_activate() {
    $queue = new MessageQueue();
    return $queue->activate();
}

function whmcs_message_queue_publish($queue, $message, $options = []) {
    $queue = new MessageQueue();
    return $queue->publish($queue, $message, $options);
}

function whmcs_message_queue_subscribe($queue, $callback) {
    $queue = new MessageQueue();
    return $queue->subscribe($queue, $callback);
}
```

### lib/MessageQueue.php

```php
<?php
namespace WHMCS\Module\MessageQueue;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class MessageQueue {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Message Queue module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_queue_messages` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `queue_name` VARCHAR(100) NOT NULL,
                `message` JSON NOT NULL,
                `priority` INT NOT NULL DEFAULT 0,
                `status` ENUM('pending', 'processing', 'completed', 'failed', 'dead') NOT NULL DEFAULT 'pending',
                `attempts` INT UNSIGNED NOT NULL DEFAULT 0,
                `max_attempts` INT UNSIGNED NOT NULL DEFAULT 3,
                `scheduled_at` DATETIME NULL,
                `processed_at` DATETIME NULL,
                `error_message` TEXT NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                INDEX `idx_queue_status` (`queue_name`, `status`, `priority`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_queue_subscriptions` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `queue_name` VARCHAR(100) NOT NULL,
                `handler` VARCHAR(255) NOT NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function publish($queueName, $message, $options = []) {
        $id = Capsule::table('mod_queue_messages')->insertGetId([
            'queue_name' => $queueName,
            'message' => json_encode($message),
            'priority' => $options['priority'] ?? 0,
            'max_attempts' => $options['max_attempts'] ?? 3,
            'scheduled_at' => $options['delay'] ?? null,
        ]);
        
        return ['success' => true, 'message_id' => $id];
    }
    
    public function subscribe($queueName, $handler) {
        Capsule::table('mod_queue_subscriptions')->insert([
            'queue_name' => $queueName,
            'handler' => $handler,
        ]);
        
        return ['success' => true];
    }
    
    public function processQueue($queueName, $limit = 100) {
        $messages = Capsule::table('mod_queue_messages')
            ->where('queue_name', $queueName)
            ->where('status', 'pending')
            ->where(function($q) {
                $q->whereNull('scheduled_at')
                    ->orWhere('scheduled_at', '<=', Carbon::now());
            })
            ->orderBy('priority', 'desc')
            ->orderBy('created_at', 'asc')
            ->limit($limit)
            ->get();
        
        $processed = 0;
        foreach ($messages as $message) {
            $this->processMessage($message);
            $processed++;
        }
        
        return ['processed' => $processed];
    }
    
    protected function processMessage($message) {
        Capsule::table('mod_queue_messages')
            ->where('id', $message->id)
            ->update(['status' => 'processing', 'attempts' => Capsule::raw('attempts + 1')]);
        
        try {
            $handlers = Capsule::table('mod_queue_subscriptions')
                ->where('queue_name', $message->queue_name)
                ->where('is_active', 1)
                ->get();
            
            foreach ($handlers as $handler) {
                $data = json_decode($message->message, true);
                call_user_func($handler->handler, $data);
            }
            
            Capsule::table('mod_queue_messages')
                ->where('id', $message->id)
                ->update(['status' => 'completed', 'processed_at' => Carbon::now()]);
                
        } catch (\Exception $e) {
            $newStatus = ($message->attempts + 1 >= $message->max_attempts) ? 'dead' : 'pending';
            
            Capsule::table('mod_queue_messages')
                ->where('id', $message->id)
                ->update([
                    'status' => $newStatus,
                    'error_message' => $e->getMessage(),
                ]);
        }
    }
}
```

add_hook('DailyCronJob', 1, function() {
    $queue = new \WHMCS\Module\MessageQueue\MessageQueue();
    $queue->processQueue('default', 50);
    $queue->processQueue('notifications', 100);
    $queue->processQueue('webhooks', 50);
});

## API Endpoints

```
POST /api/v1/queue/publish              - Publish message
GET  /api/v1/queue/{name}/messages       - Get queue messages
POST /api/v1/queue/{name}/process        - Process queue
DELETE /api/v1/queue/messages/{id}      - Delete message
```
