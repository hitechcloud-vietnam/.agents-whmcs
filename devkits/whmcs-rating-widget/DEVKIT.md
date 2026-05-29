# WHMCS Rating Widget DevKit

## Overview

Rating and feedback widget system for WHMCS enabling users to rate services, products, and experiences with star ratings, NPS surveys, and sentiment analysis.

## Features

- Star ratings
- NPS scoring
- Sentiment analysis
- Service ratings
- Product ratings
- Rating analytics
- Review display
- Widget customization
- Weighted averages

## Module Files

```php
<?php
/**
 * WHMCS Rating Widget Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/RatingWidget.php';

function whmcs_rating_widget_activate() {
    $widget = new RatingWidget();
    return $widget->activate();
}

function whmcs_rating_submit($entityType, $entityId, $userId, $rating, $review = null) {
    $widget = new RatingWidget();
    return $widget->submitRating($entityType, $entityId, $userId, $rating, $review);
}

function whmcs_rating_get($entityType, $entityId) {
    $widget = new RatingWidget();
    return $widget->getRating($entityType, $entityId);
}
```

### lib/RatingWidget.php

```php
<?php
namespace WHMCS\Module\RatingWidget;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class RatingWidget {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Rating Widget module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_ratings` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `entity_type` VARCHAR(50) NOT NULL,
                `entity_id` INT UNSIGNED NOT NULL,
                `user_id` INT UNSIGNED NOT NULL,
                `rating` TINYINT UNSIGNED NOT NULL,
                `review` TEXT NULL,
                `sentiment_score` DECIMAL(3,2) NULL,
                `is_verified_purchase` TINYINT(1) NOT NULL DEFAULT 0,
                `helpful_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `is_published` TINYINT(1) NOT NULL DEFAULT 0,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_entity_user` (`entity_type`, `entity_id`, `user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function submitRating($entityType, $entityId, $userId, $rating, $review = null) {
        Capsule::table('mod_ratings')->updateOrInsert(
            ['entity_type' => $entityType, 'entity_id' => $entityId, 'user_id' => $userId],
            [
                'rating' => $rating,
                'review' => $review,
                'sentiment_score' => $this->analyzeSentiment($review),
            ]
        );
        
        return ['success' => true];
    }
    
    public function getRating($entityType, $entityId) {
        $ratings = Capsule::table('mod_ratings')
            ->where('entity_type', $entityType)
            ->where('entity_id', $entityId)
            ->where('is_published', 1)
            ->get();
        
        if ($ratings->isEmpty()) {
            return ['average' => 0, 'count' => 0, 'distribution' => []];
        }
        
        $avgRating = $ratings->avg('rating');
        $distribution = [];
        
        for ($i = 1; $i <= 5; $i++) {
            $distribution[$i] = $ratings->where('rating', $i)->count();
        }
        
        return [
            'average' => round($avgRating, 1),
            'count' => $ratings->count(),
            'distribution' => $distribution,
        ];
    }
    
    protected function analyzeSentiment($text) {
        if (!$text) return null;
        
        $positive = ['great', 'excellent', 'amazing', 'love', 'good', 'best', 'fantastic'];
        $negative = ['bad', 'terrible', 'awful', 'hate', 'worst', 'poor', 'horrible'];
        
        $text = strtolower($text);
        $positiveCount = 0;
        $negativeCount = 0;
        
        foreach ($positive as $word) {
            $positiveCount += substr_count($text, $word);
        }
        foreach ($negative as $word) {
            $negativeCount += substr_count($text, $word);
        }
        
        $total = $positiveCount + $negativeCount;
        if ($total == 0) return 0.5;
        
        return ($positiveCount / $total);
    }
}
```

## API Endpoints

```
POST /api/v1/ratings                      - Submit rating
GET  /api/v1/ratings/{type}/{id}         - Get rating
GET  /api/v1/ratings/{type}/{id}/reviews  - Get reviews
POST /api/v1/ratings/{id}/helpful        - Mark helpful
```
