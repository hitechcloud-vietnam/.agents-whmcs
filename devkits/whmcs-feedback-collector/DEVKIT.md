# WHMCS Feedback Collector DevKit

## Overview

Feedback collection system for WHMCS enabling gathering, organizing, and analyzing customer feedback across multiple touchpoints.

## Features

- Multi-channel feedback
- Feedback categories
- Priority scoring
- Response tracking
- Satisfaction metrics
- Feedback trends
- Automated actions

## Module Files

```php
<?php
/**
 * WHMCS Feedback Collector Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/FeedbackCollector.php';

function whmcs_feedback_collector_activate() {
    $collector = new FeedbackCollector();
    return $collector->activate();
}

function whmcs_feedback_submit($userId, $category, $feedback, $rating = null) {
    $collector = new FeedbackCollector();
    return $collector->submitFeedback($userId, $category, $feedback, $rating);
}
```

### lib/FeedbackCollector.php

```php
<?php
namespace WHMCS\Module\FeedbackCollector;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class FeedbackCollector {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Feedback Collector module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_feedback` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `feedback_code` VARCHAR(50) NOT NULL,
                `user_id` INT UNSIGNED NOT NULL,
                `category` VARCHAR(100) NOT NULL,
                `feedback` TEXT NOT NULL,
                `rating` TINYINT UNSIGNED NULL,
                `sentiment` ENUM('positive', 'neutral', 'negative') NULL,
                `status` ENUM('new', 'reviewed', 'actioned', 'archived') NOT NULL DEFAULT 'new',
                `assigned_to` INT UNSIGNED NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function submitFeedback($userId, $category, $feedback, $rating = null) {
        $id = Capsule::table('mod_feedback')->insertGetId([
            'feedback_code' => 'FB-' . strtoupper(substr(uniqid(), -8)),
            'user_id' => $userId,
            'category' => $category,
            'feedback' => $feedback,
            'rating' => $rating,
        ]);
        
        return ['success' => true, 'feedback_id' => $id];
    }
    
    public function getAnalytics($days = 30) {
        return [
            'total' => Capsule::table('mod_feedback')
                ->where('created_at', '>=', Carbon::now()->subDays($days))
                ->count(),
            'by_sentiment' => Capsule::table('mod_feedback')
                ->where('created_at', '>=', Carbon::now()->subDays($days))
                ->groupBy('sentiment')
                ->selectRaw('sentiment, COUNT(*) as count')
                ->get(),
        ];
    }
}
```

## API Endpoints

```
POST /api/v1/feedback                     - Submit feedback
GET  /api/v1/feedback                  - List feedback
GET  /api/v1/feedback/analytics         - Get analytics
```
