# WHMCS Review Moderation DevKit

## Overview

Review moderation system for WHMCS enabling filtering, approval workflows, and content management for user reviews.

## Module Files

```php
<?php
/**
 * WHMCS Review Moderation Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/ReviewModeration.php';

function whmcs_review_moderation_activate() {
    $mod = new ReviewModeration();
    return $mod->activate();
}

function whmcs_review_moderation_submit($data) {
    $mod = new ReviewModeration();
    return $mod->submitReview($data);
}

function whmcs_review_moderation_approve($reviewId) {
    $mod = new ReviewModeration();
    return $mod->approveReview($reviewId);
}
```

### lib/ReviewModeration.php

```php
<?php
namespace WHMCS\Module\ReviewModeration;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class ReviewModeration {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Review Moderation module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_reviews` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `review_code` VARCHAR(50) NOT NULL,
                `entity_type` VARCHAR(50) NOT NULL,
                `entity_id` INT UNSIGNED NOT NULL,
                `user_id` INT UNSIGNED NOT NULL,
                `rating` TINYINT UNSIGNED NOT NULL,
                `title` VARCHAR(255) NULL,
                `content` TEXT NOT NULL,
                `status` ENUM('pending', 'approved', 'rejected', 'flagged') NOT NULL DEFAULT 'pending',
                `moderation_notes` TEXT NULL,
                `moderated_by` INT UNSIGNED NULL,
                `moderated_at` DATETIME NULL,
                `helpful_count` INT UNSIGNED DEFAULT 0,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_moderation_rules` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `rule_name` VARCHAR(255) NOT NULL,
                `rule_type` VARCHAR(50) NOT NULL,
                `conditions` JSON NOT NULL,
                `action` ENUM('auto_approve', 'auto_reject', 'flag_for_review') NOT NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function submitReview($data) {
        $id = Capsule::table('mod_reviews')->insertGetId([
            'review_code' => 'REV-' . strtoupper(substr(uniqid(), -8)),
            'entity_type' => $data['entity_type'],
            'entity_id' => $data['entity_id'],
            'user_id' => $data['user_id'],
            'rating' => $data['rating'],
            'title' => $data['title'] ?? null,
            'content' => $data['content'],
        ]);
        
        // Auto-moderation
        $this->applyAutoModeration($id);
        
        return ['success' => true, 'review_id' => $id];
    }
    
    protected function applyAutoModeration($reviewId) {
        $review = Capsule::table('mod_reviews')->where('id', $reviewId)->first();
        $rules = Capsule::table('mod_moderation_rules')->where('is_active', 1)->get();
        
        foreach ($rules as $rule) {
            $conditions = json_decode($rule->conditions, true);
            
            // Simple keyword check
            if (!empty($conditions['blocked_words'])) {
                foreach ($conditions['blocked_words'] as $word) {
                    if (stripos($review->content, $word) !== false) {
                        Capsule::table('mod_reviews')
                            ->where('id', $reviewId)
                            ->update(['status' => 'rejected']);
                        return;
                    }
                }
            }
        }
    }
    
    public function approveReview($reviewId, $moderatorId) {
        Capsule::table('mod_reviews')
            ->where('id', $reviewId)
            ->update([
                'status' => 'approved',
                'moderated_by' => $moderatorId,
                'moderated_at' => Carbon::now(),
            ]);
        
        return ['success' => true];
    }
    
    public function getPendingReviews() {
        return Capsule::table('mod_reviews')
            ->where('status', 'pending')
            ->orderBy('created_at', 'asc')
            ->get();
    }
}
```

## API Endpoints

```
POST /api/v1/reviews/submit               - Submit review
GET  /api/v1/reviews/pending             - Get pending reviews
POST /api/v1/reviews/{id}/approve        - Approve
POST /api/v1/reviews/{id}/reject         - Reject
```
