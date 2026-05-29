# WHMCS Rate Limiting Skill

## Purpose
Provides patterns for implementing rate limiting in WHMCS, controlling API usage, managing request quotas, and protecting against abuse.

## Implementation Patterns

### Rate Limiter
```php
<?php
class RateLimiter {
    private $db;
    private $cache;
    
    public function __construct() {
        $this->cache = \WHMCS\Application\Services\CacheService::getInstance();
    }
    
    public function check($identifier, $limitType = 'api', $increment = 1) {
        $limits = $this->getLimits($limitType);
        
        $key = "rate_limit_{$limitType}_{$identifier}";
        $current = (int)$this->cache->get($key) ?: 0;
        
        if ($current >= $limits['max_requests']) {
            return [
                'allowed' => false,
                'limit' => $limits['max_requests'],
                'remaining' => 0,
                'reset_at' => $this->getResetTime($limits['window_seconds'])
            ];
        }
        
        if ($current == 0) {
            $this->cache->set($key, $increment, $limits['window_seconds']);
        } else {
            $this->cache->increment($key, $increment);
        }
        
        return [
            'allowed' => true,
            'limit' => $limits['max_requests'],
            'remaining' => $limits['max_requests'] - $current - $increment,
            'reset_at' => $this->getResetTime($limits['window_seconds'])
        ];
    }
    
    public function getLimits($limitType) {
        $limits = [
            'api' => ['max_requests' => 1000, 'window_seconds' => 3600],
            'login' => ['max_requests' => 10, 'window_seconds' => 300],
            'payment' => ['max_requests' => 5, 'window_seconds' => 60]
        ];
        
        return $limits[$limitType] ?? $limits['api'];
    }
    
    public function setLimits($limitType, $maxRequests, $windowSeconds) {
        $this->db->insert('mod_rate_limits', [
            'limit_type' => $limitType,
            'max_requests' => $maxRequests,
            'window_seconds' => $windowSeconds,
            'updated_at' => date('Y-m-d H:i:s')
        ], true);
    }
    
    public function reset($identifier, $limitType) {
        $key = "rate_limit_{$limitType}_{$identifier}";
        $this->cache->delete($key);
    }
    
    private function getResetTime($windowSeconds) {
        return time() + $windowSeconds;
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_rate_limits (
    id INT AUTO_INCREMENT PRIMARY KEY,
    limit_type VARCHAR(50),
    max_requests INT,
    window_seconds INT,
    updated_at DATETIME
);

CREATE TABLE mod_rate_limit_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    identifier VARCHAR(255),
    limit_type VARCHAR(50),
    allowed TINYINT(1),
    created_at DATETIME
);
```

## Usage Examples
```php
$limiter = new RateLimiter();
$result = $limiter->check($clientId, 'api');

if (!$result['allowed']) {
    http_response_code(429);
    header('Retry-After: ' . $result['reset_at']);
    die('Rate limit exceeded');
}
```
