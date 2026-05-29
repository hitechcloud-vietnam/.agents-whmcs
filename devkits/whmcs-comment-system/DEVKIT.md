# WHMCS Comment System DevKit

## Overview

Comment system for WHMCS enabling threaded comments on articles, products, and services.

## Module Files

```php
<?php
/**
 * WHMCS Comment System Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/CommentSystem.php';

function whmcs_comment_system_activate() {
    $comments = new CommentSystem();
    return $comments->activate();
}

function whmcs_comment_create($data) {
    $comments = new CommentSystem();
    return $comments->createComment($data);
}
```

### lib/CommentSystem.php

```php
<?php
namespace WHMCS\Module\CommentSystem;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class CommentSystem {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Comment System module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_comments` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `entity_type` VARCHAR(50) NOT NULL,
                `entity_id` INT UNSIGNED NOT NULL,
                `parent_id` INT UNSIGNED NULL,
                `user_id` INT UNSIGNED NOT NULL,
                `content` TEXT NOT NULL,
                `status` ENUM('pending', 'approved', 'deleted') NOT NULL DEFAULT 'approved',
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function createComment($data) {
        $id = Capsule::table('mod_comments')->insertGetId([
            'entity_type' => $data['entity_type'],
            'entity_id' => $data['entity_id'],
            'parent_id' => $data['parent_id'] ?? null,
            'user_id' => $data['user_id'],
            'content' => $data['content'],
        ]);
        
        return ['success' => true, 'comment_id' => $id];
    }
    
    public function getComments($entityType, $entityId) {
        return Capsule::table('mod_comments as c')
            ->leftJoin('tblusers as u', 'c.user_id', '=', 'u.id')
            ->where('c.entity_type', $entityType)
            ->where('c.entity_id', $entityId)
            ->where('c.status', 'approved')
            ->whereNull('c.parent_id')
            ->select('c.*', 'u.firstname', 'u.lastname')
            ->orderBy('c.created_at', 'desc')
            ->get();
    }
}
```

## API Endpoints

```
POST /api/v1/comments                     - Create comment
GET  /api/v1/comments/{type}/{id}       - Get comments
DELETE /api/v1/comments/{id}            - Delete comment
```
