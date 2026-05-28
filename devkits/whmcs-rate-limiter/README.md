# WHMCS Rate Limiter Module

API rate limiting module with configurable limits, multiple strategies, and comprehensive analytics.

## Features

- Multiple rate limiting strategies (Fixed, Sliding, Token Bucket)
- Per-IP, per-API key, per-user limiting
- Per-endpoint limiting
- Whitelist support
- Comprehensive logging
- Usage statistics
- Automatic cleanup

## Installation

Copy module to `/path/to/whmcs/modules/servers/ratelimiter/` and activate.

## Usage

```php
// Check if request is allowed
$result = ratelimiter_IsAllowed($ip, 'ip', '/api/v1/users');

if (!$result['allowed']) {
    header('Retry-After: ' . $result['retry_after']);
    http_response_code(429);
    exit('Rate limit exceeded');
}

// Set custom limit
ratelimiter_SetLimit('api_key_123', 'api_key', 1000, 3600);

// Get stats
$stats = ratelimiter_GetStats('192.168.1.1', 7);
```

## API Functions

| Function | Description |
|----------|-------------|
| `ratelimiter_SetLimit()` | Set rate limit |
| `ratelimiter_GetLimit()` | Get limit config |
| `ratelimiter_IsAllowed()` | Check request |
| `ratelimiter_GetStats()` | Get statistics |
| `ratelimiter_CleanOldData()` | Cleanup old data |
