# WHMCS API Rate Limiter DevKit

## Overview

API rate limiting system for WHMCS that controls API usage per user/tenant, implements various rate limiting strategies, handles burst traffic, and provides rate limit headers.

## Features

- Per-user rate limiting
- Per-tenant rate limiting
- Multiple algorithms (token bucket, sliding window)
- Burst handling
- Rate limit headers
- Custom rate limits
- Tiered pricing
- Quota tracking

## Module Files

```php
<?php
/**
 * WHMCS API Rate Limiter Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/RateLimiter.php';

function whmcs_api_rate_limiter_check($userId, $endpoint) {
    $limiter = new RateLimiter();
    return $limiter->check($userId, $endpoint);
}

function whmcs_api_rate_limiter_get_headers($userId) {
    $limiter = new RateLimiter();
    return $limiter->getHeaders($userId);
}

add_hook('APIBeforeCall', 1, function($params) {
    $limiter = new RateLimiter();
    $result = $limiter->check($params['user_id'] ?? 0, $params['endpoint'] ?? '');
    
    if (!$result['allowed']) {
        http_response_code(429);
        header('Retry-After: ' . $result['retry_after']);
        die(json_encode(['error' => 'Rate limit exceeded', 'retry_after' => $result['retry_after']]));
    }
});
```

### lib/RateLimiter.php

```php
<?php
namespace WHMCS\Module\ApiRateLimiter;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class RateLimiter {
    
    protected $defaultLimits = [
        'default' => ['requests' => 100, 'window' => 60],
        'authenticated' => ['requests' => 1000, 'window' => 60],
        'premium' => ['requests' => 10000, 'window' => 60],
    ];
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'API Rate Limiter module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_rate_limits` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `user_id` INT UNSIGNED NOT NULL,
                `endpoint` VARCHAR(255) NULL,
                `requests_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `window_start` DATETIME NOT NULL,
                `window_duration` INT UNSIGNED NOT NULL DEFAULT 60,
                `max_requests` INT UNSIGNED NOT NULL DEFAULT 100,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_user_endpoint` (`user_id`, `endpoint`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_rate_limit_configs` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `name` VARCHAR(100) NOT NULL,
                `requests_per_window` INT UNSIGNED NOT NULL,
                `window_duration` INT UNSIGNED NOT NULL,
                `limit_type` VARCHAR(30) NOT NULL DEFAULT 'user',
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function check($userId, $endpoint = null) {
        $limit = $this->getUserLimit($userId);
        $windowDuration = $limit['window'] ?? 60;
        $maxRequests = $limit['requests'] ?? 100;
        
        $rateLimit = Capsule::table('mod_rate_limits')
            ->where('user_id', $userId)
            ->where(function($q) use ($endpoint) {
                $q->where('endpoint', $endpoint ?? '')
                    ->orWhereNull('endpoint');
            })
            ->first();
        
        if (!$rateLimit) {
            Capsule::table('mod_rate_limits')->insert([
                'user_id' => $userId,
                'endpoint' => $endpoint,
                'requests_count' => 1,
                'window_start' => Carbon::now(),
                'window_duration' => $windowDuration,
                'max_requests' => $maxRequests,
            ]);
            
            return [
                'allowed' => true,
                'remaining' => $maxRequests - 1,
                'limit' => $maxRequests,
            ];
        }
        
        $windowStart = Carbon::parse($rateLimit->window_start);
        $windowEnd = $windowStart->copy()->addSeconds($rateLimit->window_duration);
        
        if (Carbon::now()->gte($windowEnd)) {
            Capsule::table('mod_rate_limits')
                ->where('id', $rateLimit->id)
                ->update([
                    'requests_count' => 1,
                    'window_start' => Carbon::now(),
                ]);
            
            return [
                'allowed' => true,
                'remaining' => $maxRequests - 1,
                'limit' => $maxRequests,
                'reset_at' => Carbon::now()->addSeconds($windowDuration),
            ];
        }
        
        if ($rateLimit->requests_count >= $maxRequests) {
            $retryAfter = Carbon::now()->diffInSeconds($windowEnd);
            
            return [
                'allowed' => false,
                'remaining' => 0,
                'limit' => $maxRequests,
                'retry_after' => $retryAfter,
            ];
        }
        
        Capsule::table('mod_rate_limits')
            ->where('id', $rateLimit->id)
            ->increment('requests_count');
        
        return [
            'allowed' => true,
            'remaining' => $maxRequests - $rateLimit->requests_count - 1,
            'limit' => $maxRequests,
            'reset_at' => $windowEnd,
        ];
    }
    
    protected function getUserLimit($userId) {
        if (!$userId) return $this->defaultLimits['default'];
        
        $client = Capsule::table('tblclients')->where('id', $userId)->first();
        
        if (!$client) return $this->defaultLimits['authenticated'];
        
        $service = Capsule::table('tblhosting')
            ->where('userid', $userId)
            ->where('domainstatus', 'Active')
            ->first();
        
        if ($service) {
            return $this->defaultLimits['premium'];
        }
        
        return $this->defaultLimits['authenticated'];
    }
    
    public function getHeaders($userId) {
        $limit = $this->getUserLimit($userId);
        
        return [
            'X-RateLimit-Limit' => $limit['requests'],
            'X-RateLimit-Remaining' => $this->getRemaining($userId),
            'X-RateLimit-Reset' => Carbon::now()->addSeconds($limit['window'])->timestamp,
        ];
    }
    
    protected function getRemaining($userId) {
        $rateLimit = Capsule::table('mod_rate_limits')
            ->where('user_id', $userId)
            ->first();
        
        if (!$rateLimit) {
            return $this->getUserLimit($userId)['requests'];
        }
        
        return max(0, $rateLimit->max_requests - $rateLimit->requests_count);
    }
}
```

## API Endpoints

```
POST /api/v1/rate-limiter/check        - Check rate limit
GET  /api/v1/rate-limiter/headers      - Get rate limit headers
PUT  /api/v1/rate-limiter/config       - Update limit config
GET  /api/v1/rate-limiter/stats        - Get usage stats
```
