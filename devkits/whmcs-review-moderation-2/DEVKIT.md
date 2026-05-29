# WHMCS Review Moderation DevKit

## Overview

Review moderation system for WHMCS enabling filtering, approval workflows, and content management for user reviews.

## Features

- Content moderation
- Auto-filtering
- Manual review queue
- Spam protection
- Profanity filtering
- Quality scoring

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
            CREATE TABLE IF NOT EXISTS `mod_review_moderation` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `review_id` VARCHAR(64) NOT NULL,
                `entity_type` VARCHAR(50) NOT NULL,
                `entity_id` INT UNSIGNED NOT NULL,
                `user_id` INT UNSIGNED NOT NULL,
                `rating` TINYINT UNSIGNED NOT NULL,
                `content` TEXT NOT NULL,
                `quality_score` DECIMAL(5,2) DEFAULT 0,
                `status` ENUM('pending', 'approved', 'rejected', 'flagged') NOT NULL DEFAULT 'pending',
                `moderation_notes` TEXT NULL,
                `moderated_by` INT UNSIGNED NULL,
                `moderated_at` DATETIME NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_moderation_filters` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `filter_name` VARCHAR(255) NOT NULL,
                `filter_type` ENUM('profanity', 'spam', 'link', 'keyword') NOT NULL,
                `pattern` TEXT NOT NULL,
                `action` ENUM('auto_approve', 'auto_reject', 'flag') NOT NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function submitReview($data) {
        $reviewId = 'REV-' . strtoupper(substr(uniqid(), 0, 12));
        
        // Calculate quality score
        $qualityScore = $this->calculateQualityScore($data['content']);
        
        // Apply filters
        $filterResult = $this->applyFilters($data['content']);
        
        $status = $filterResult['action'];
        
        $id = Capsule::table('mod_review_moderation')->insertGetId([
            'review_id' => $reviewId,
            'entity_type' => $data['entity_type'],
            'entity_id' => $data['entity_id'],
            'user_id' => $data['user_id'],
            'rating' => $data['rating'],
            'content' => $data['content'],
            'quality_score' => $qualityScore,
            'status' => $status,
        ]);
        
        return ['success' => true, 'review_id' => $reviewId, 'status' => $status];
    }
    
    protected function calculateQualityScore($content) {
        $score = 50; // Base score
        
        // Factor in content length
        if (strlen($content) > 100) $score += 20;
        if (strlen($content) > 300) $score += 20;
        
        // Factor in sentence variety
        $sentences = preg_split('/[.!?]+/', $content);
        if (count($sentences) > 3) $score += 10;
        
        return min(100, $score);
    }
    
    protected function applyFilters($content) {
        $filters = Capsule::table('mod_moderation_filters')->where('is_active', 1)->get();
        
        foreach ($filters as $filter) {
            if (preg_match('/' . $filter->pattern . '/i', $content)) {
                return ['action' => $filter->action, 'filter' => $filter->filter_name];
            }
        }
        
        return ['action' => 'pending'];
    }
    
    public function approveReview($reviewId, $moderatorId) {
        Capsule::table('mod_review_moderation')
            ->where('review_id', $reviewId)
            ->update([
                'status' => 'approved',
                'moderated_by' => $moderatorId,
                'moderated_at' => Carbon::now(),
            ]);
        
        return ['success' => true];
    }
    
    public function getPendingReviews() {
        return Capsule::table('mod_review_moderation')
            ->where('status', 'pending')
            ->orderBy('created_at', 'asc')
            ->get();
    }
}
```

## API Endpoints

```
POST /api/v1/reviews/submit               - Submit review
GET  /api/v1/reviews/pending            - Get pending reviews
POST /api/v1/reviews/{id}/approve       - Approve review
POST /api/v1/reviews/{id}/reject        - Reject review
```
