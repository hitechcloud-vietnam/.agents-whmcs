# WHMCS Usage Analytics DevKit

## Overview

Comprehensive usage analytics and tracking system for WHMCS that captures user behavior, feature usage patterns, API call metrics, session analytics, and generates actionable insights.

## Features

- User behavior tracking
- Feature usage analytics
- API call metrics
- Session analytics
- Heat maps
- Funnel analysis
- Custom event tracking
- Real-time dashboards
- Export capabilities
- Cohort comparison

## Module Files

```php
<?php
/**
 * WHMCS Usage Analytics Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/AnalyticsEngine.php';
require_once __DIR__ . '/lib/EventTracker.php';

/**
 * Track an analytics event
 */
function whmcs_usage_analytics_track($eventName, $properties = [], $userId = null) {
    $tracker = new EventTracker();
    return $tracker->track($eventName, $properties, $userId);
}

/**
 * Get analytics dashboard
 */
function whmcs_usage_analytics_dashboard($options = []) {
    $engine = new AnalyticsEngine();
    return $engine->getDashboard($options);
}

/**
 * Get funnel analysis
 */
function whmcs_usage_analytics_funnel($funnelId) {
    $engine = new AnalyticsEngine();
    return $engine->getFunnelAnalysis($funnelId);
}

add_hook('ClientAreaPage下班', 1, function($params) {
    $tracker = new EventTracker();
    $tracker->track('page_view', [
        'page' => $params['filename'],
        'title' => $params['pagetitle'] ?? '',
    ], $params['user_id']);
});

add_hook('DailyCronJob', 1, function() {
    $engine = new AnalyticsEngine();
    $engine->aggregateDailyStats();
});
```

### lib/AnalyticsEngine.php

```php
<?php
namespace WHMCS\Module\UsageAnalytics;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class AnalyticsEngine {
    
    public function getDashboard($options = []) {
        $period = $options['period'] ?? '7d';
        $startDate = $this->getPeriodStart($period);
        
        return [
            'overview' => $this->getOverviewMetrics($startDate),
            'page_views' => $this->getPageViews($startDate),
            'top_features' => $this->getTopFeatures($startDate),
            'user_activity' => $this->getUserActivity($startDate),
            'api_usage' => $this->getApiUsage($startDate),
        ];
    }
    
    protected function getPeriodStart($period) {
        $map = [
            '24h' => Carbon::now()->subDay(),
            '7d' => Carbon::now()->subDays(7),
            '30d' => Carbon::now()->subDays(30),
            '90d' => Carbon::now()->subDays(90),
        ];
        return $map[$period] ?? Carbon::now()->subDays(7);
    }
    
    protected function getOverviewMetrics($startDate) {
        $stats = Capsule::table('mod_analytics_events')
            ->where('created_at', '>=', $startDate)
            ->selectRaw('COUNT(*) as total_events, COUNT(DISTINCT user_id) as unique_users')
            ->first();
        
        $sessions = Capsule::table('mod_analytics_sessions')
            ->where('started_at', '>=', $startDate)
            ->count();
        
        return [
            'total_events' => $stats->total_events ?? 0,
            'unique_users' => $stats->unique_users ?? 0,
            'sessions' => $sessions,
            'avg_session_duration' => $this->getAvgSessionDuration($startDate),
        ];
    }
    
    protected function getAvgSessionDuration($startDate) {
        $result = Capsule::table('mod_analytics_sessions')
            ->where('started_at', '>=', $startDate)
            ->whereNotNull('ended_at')
            ->selectRaw('AVG(TIMESTAMPDIFF(SECOND, started_at, ended_at)) as avg_duration')
            ->first();
        
        return round($result->avg_duration ?? 0);
    }
    
    protected function getTopFeatures($startDate) {
        return Capsule::table('mod_analytics_events')
            ->where('created_at', '>=', $startDate)
            ->whereNotNull('event_name')
            ->groupBy('event_name')
            ->selectRaw('event_name, COUNT(*) as count')
            ->orderBy('count', 'desc')
            ->limit(10)
            ->get();
    }
    
    protected function getApiUsage($startDate) {
        return Capsule::table('mod_analytics_api_calls')
            ->where('created_at', '>=', $startDate)
            ->selectRaw('endpoint, COUNT(*) as calls, AVG(response_time_ms) as avg_response_time')
            ->groupBy('endpoint')
            ->orderBy('calls', 'desc')
            ->limit(10)
            ->get();
    }
    
    public function getFunnelAnalysis($funnelId) {
        $funnel = Capsule::table('mod_analytics_funnels')->where('id', $funnelId)->first();
        
        if (!$funnel) return null;
        
        $steps = json_decode($funnel->steps, true);
        $results = [];
        
        foreach ($steps as $index => $step) {
            $count = Capsule::table('mod_analytics_events')
                ->where('event_name', $step['event'])
                ->where('created_at', '>=', Carbon::now()->subDays(30))
                ->count();
            
            $conversionRate = $index > 0 && isset($results[$index - 1]['count']) && $results[$index - 1]['count'] > 0
                ? ($count / $results[$index - 1]['count']) * 100
                : 100;
            
            $results[] = [
                'step' => $index + 1,
                'name' => $step['name'],
                'event' => $step['event'],
                'count' => $count,
                'conversion_rate' => round($conversionRate, 2),
            ];
        }
        
        return $results;
    }
    
    public function aggregateDailyStats() {
        $yesterday = Carbon::yesterday();
        
        // Aggregate page views
        $pageViews = Capsule::table('mod_analytics_events')
            ->where('created_at', '>=', $yesterday->copy()->startOfDay())
            ->where('created_at', '<', $yesterday->copy()->endOfDay())
            ->where('event_name', 'page_view')
            ->count();
        
        Capsule::table('mod_analytics_daily_stats')->updateOrInsert(
            ['stat_date' => $yesterday->toDateString(), 'stat_type' => 'page_views'],
            ['stat_value' => $pageViews]
        );
    }
}

class EventTracker {
    
    public function track($eventName, $properties = [], $userId = null) {
        if (!$userId) {
            $userId = $_SESSION['uid'] ?? null;
        }
        
        $eventId = Capsule::table('mod_analytics_events')->insertGetId([
            'event_name' => $eventName,
            'properties' => json_encode($properties),
            'user_id' => $userId,
            'session_id' => session_id(),
            'ip_address' => $this->getClientIp(),
            'user_agent' => substr($_SERVER['HTTP_USER_AGENT'] ?? '', 0, 500),
        ]);
        
        // Track session if not exists
        $this->ensureSession();
        
        return ['success' => true, 'event_id' => $eventId];
    }
    
    protected function ensureSession() {
        $exists = Capsule::table('mod_analytics_sessions')
            ->where('session_id', session_id())
            ->where('ended_at', null)
            ->exists();
        
        if (!$exists) {
            Capsule::table('mod_analytics_sessions')->insert([
                'session_id' => session_id(),
                'user_id' => $_SESSION['uid'] ?? null,
                'started_at' => Carbon::now(),
                'ip_address' => $this->getClientIp(),
            ]);
        }
    }
    
    protected function getClientIp() {
        $headers = ['HTTP_CF_CONNECTING_IP', 'HTTP_X_FORWARDED_FOR', 'REMOTE_ADDR'];
        foreach ($headers as $header) {
            if (!empty($_SERVER[$header])) {
                return $_SERVER[$header];
            }
        }
        return '0.0.0.0';
    }
}
```

## Database Tables (to be created on activate)

```sql
CREATE TABLE IF NOT EXISTS `mod_analytics_events` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `event_name` VARCHAR(100) NOT NULL,
    `properties` JSON NULL,
    `user_id` INT UNSIGNED NULL,
    `session_id` VARCHAR(128) NULL,
    `ip_address` VARCHAR(45) NULL,
    `user_agent` VARCHAR(500) NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_event_date` (`event_name`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_analytics_sessions` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `session_id` VARCHAR(128) NOT NULL,
    `user_id` INT UNSIGNED NULL,
    `started_at` DATETIME NOT NULL,
    `ended_at` DATETIME NULL,
    `ip_address` VARCHAR(45) NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_session` (`session_id`, `started_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## API Endpoints

```
POST /api/v1/analytics/track            - Track event
GET  /api/v1/analytics/dashboard        - Get dashboard
GET  /api/v1/analytics/funnels/{id}    - Get funnel analysis
GET  /api/v1/analytics/top-features     - Get top features
GET  /api/v1/analytics/user/{id}/events - Get user events
```
