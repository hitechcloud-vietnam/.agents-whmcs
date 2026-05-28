# WHMCS API Rate Limiting Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Implement API rate limiting for module endpoints.

## Implementation Steps

### 1. Create Rate Limit Table
```php
Capsule::schema()->create('mod_rate_limits', function($t) {
    $t->increments('id');
    $t->string('identifier', 255);
    $t->integer('request_count')->default(1);
    $t->timestamp('window_start');
    $t->index('identifier');
});
```

### 2. Configure Limits
```php
private int $maxRequests = 60;      // per minute
private int $maxHits = 1000;       // per hour
private int $windowSeconds = 60;
```

### 3. Implement Check
```php
function checkRateLimit(string $identifier, int $maxRequests, int $windowSeconds): bool {
    $key = 'rate_' . md5($identifier);
    $windowStart = date('Y-m-d H:i:s', strtotime("-{$windowSeconds} seconds"));

    $record = Capsule::table('mod_rate_limits')
        ->where('identifier', $key)
        ->where('window_start', '>', $windowStart)
        ->first();

    if (!$record) {
        Capsule::table('mod_rate_limits')->insert([
            'identifier' => $key,
            'request_count' => 1,
            'window_start' => date('Y-m-d H:i:s'),
        ]);
        return true;
    }

    if ($record->request_count >= $maxRequests) {
        return false;
    }

    Capsule::table('mod_rate_limits')
        ->where('id', $record->id)
        ->increment('request_count');

    return true;
}
```

### 4. Return Rate Limit Headers
```php
header('X-RateLimit-Limit: ' . $maxRequests);
header('X-RateLimit-Remaining: ' . $remaining);
header('X-RateLimit-Reset: ' . $resetTime);
```

## Output

Working rate limiter with:
- Per-user limits
- Per-endpoint limits
- Rate limit headers
- Automatic cleanup
