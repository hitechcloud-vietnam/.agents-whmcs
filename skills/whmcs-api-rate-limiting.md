# WHMCS API Rate Limiting

## Skill Description
Implement API rate limiting for WHMCS modules to prevent abuse, ensure fair usage, and protect server resources. This skill covers implementing rate limiting using Redis, file-based storage, and integration with WHMCS API.

## Prerequisites
- WHMCS installation with API access
- PHP 7.4+ with pcntl extension
- Redis server (optional, for distributed rate limiting) OR file system with write permissions
- Basic understanding of HTTP headers and request/response cycles

## Step-by-Step Implementation

### 1. Create Rate Limiter Class
```php
<?php
// includes/rate-limiter/RateLimiter.php

namespace WHMCS\Module\YourModule\RateLimiter;

class RateLimiter
{
    private $storage = [];
    private $maxRequests;
    private $windowSeconds;
    private $storageType; // 'redis' or 'file'
    private $redis = null;
    private $cacheDir;

    public function __construct(int $maxRequests = 60, int $windowSeconds = 60, string $storageType = 'file')
    {
        $this->maxRequests = $maxRequests;
        $this->windowSeconds = $windowSeconds;
        $this->storageType = $storageType;
        $this->cacheDir = dirname(__DIR__) . '/cache/rate_limiter/';

        if ($storageType === 'redis') {
            $this->initializeRedis();
        } else {
            $this->ensureCacheDirExists();
        }
    }

    private function initializeRedis(): void
    {
        $redisHost = defined('REDIS_HOST') ? REDIS_HOST : '127.0.0.1';
        $redisPort = defined('REDIS_PORT') ? REDIS_PORT : 6379;

        $this->redis = new \Redis();
        $this->redis->connect($redisHost, $redisPort);
    }

    private function ensureCacheDirExists(): void
    {
        if (!is_dir($this->cacheDir)) {
            mkdir($this->cacheDir, 0755, true);
        }
    }

    public function isAllowed(string $identifier): bool
    {
        $key = $this->getKey($identifier);

        if ($this->storageType === 'redis') {
            return $this->checkRedis($key);
        }

        return $this->checkFile($key);
    }

    private function getKey(string $identifier): string
    {
        return md5($identifier);
    }

    private function checkRedis(string $key): bool
    {
        $current = (int) $this->redis->get($key);

        if ($current >= $this->maxRequests) {
            return false;
        }

        $this->redis->incr($key);

        if ($current === 0) {
            $this->redis->expire($key, $this->windowSeconds);
        }

        return true;
    }

    private function checkFile(string $key): bool
    {
        $filePath = $this->cacheDir . $key . '.json';
        $now = time();

        if (file_exists($filePath)) {
            $data = json_decode(file_get_contents($filePath), true);

            // Reset if window has passed
            if ($data['timestamp'] < ($now - $this->windowSeconds)) {
                $data = ['count' => 0, 'timestamp' => $now];
            }

            if ($data['count'] >= $this->maxRequests) {
                $this->addRateLimitHeaders($data['count'], $data['timestamp']);
                return false;
            }

            $data['count']++;
        } else {
            $data = ['count' => 1, 'timestamp' => $now];
        }

        file_put_contents($filePath, json_encode($data));
        $this->addRateLimitHeaders($data['count'], $data['timestamp']);

        return true;
    }

    private function addRateLimitHeaders(int $count, int $timestamp): void
    {
        $remaining = max(0, $this->maxRequests - $count);
        $resetTime = $timestamp + $this->windowSeconds;

        header('X-RateLimit-Limit: ' . $this->maxRequests);
        header('X-RateLimit-Remaining: ' . $remaining);
        header('X-RateLimit-Reset: ' . $resetTime);
        header('Retry-After: ' . ($resetTime - time()));
    }

    public function getRemainingRequests(string $identifier): int
    {
        $key = $this->getKey($identifier);

        if ($this->storageType === 'redis') {
            $current = (int) $this->redis->get($key);
        } else {
            $filePath = $this->cacheDir . $key . '.json';
            if (file_exists($filePath)) {
                $data = json_decode(file_get_contents($filePath), true);
                $current = $data['count'];
            } else {
                $current = 0;
            }
        }

        return max(0, $this->maxRequests - $current);
    }

    public function reset(string $identifier): void
    {
        $key = $this->getKey($identifier);

        if ($this->storageType === 'redis') {
            $this->redis->del($key);
        } else {
            $filePath = $this->cacheDir . $key . '.json';
            if (file_exists($filePath)) {
                unlink($filePath);
            }
        }
    }
}
```

### 2. Create Rate Limit Middleware
```php
<?php
// includes/rate-limiter/RateLimitMiddleware.php

namespace WHMCS\Module\YourModule\RateLimiter;

use Illuminate\Http\Request;

class RateLimitMiddleware
{
    private $rateLimiter;

    public function __construct()
    {
        $this->rateLimiter = new RateLimiter(
            maxRequests: 60,
            windowSeconds: 60,
            storageType: 'file'
        );
    }

    public function handle(Request $request, \Closure $next)
    {
        $identifier = $this->getIdentifier($request);

        if (!$this->rateLimiter->isAllowed($identifier)) {
            return response()->json([
                'status' => 'error',
                'message' => 'Rate limit exceeded. Please try again later.',
                'retryAfter' => 60
            ], 429, [
                'X-RateLimit-Limit' => '60',
                'X-RateLimit-Remaining' => '0',
                'Retry-After' => '60'
            ]);
        }

        return $next($request);
    }

    private function getIdentifier(Request $request): string
    {
        // Use API key as identifier if available
        if ($request->hasHeader('X-API-Key')) {
            return 'api:' . $request->header('X-API-Key');
        }

        // Fall back to IP address
        return 'ip:' . ($request->ip() ?? 'unknown');
    }
}
```

### 3. Register Hook for API Rate Limiting
```php
<?php
// hooks.php

use WHMCS\Module\YourModule\RateLimiter\RateLimiter;
use WHMCS\Module\YourModule\RateLimiter\RateLimitMiddleware;

add_hook('PreAPI1', 1, function($vars) {
    $rateLimiter = new RateLimiter(100, 60, 'file');

    $identifier = isset($vars['identifier']) ? $vars['identifier'] :
                 ($vars['ip'] ?? 'unknown');

    if (!$rateLimiter->isAllowed('api:' . $identifier)) {
        return [
            'status' => 'error',
            'message' => 'API rate limit exceeded',
            'retryafter' => 60
        ];
    }
});
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Rate limits reset unexpectedly | Use atomic operations in Redis; use file locking for file-based storage |
| Distributed rate limiting doesn't work | Ensure Redis is shared across all servers; use consistent hashing |
| Rate limit headers missing | Call header functions before any output is sent |
| Whitelist bypass attempts | Implement IP whitelist before rate limiter check |
| Memory leaks with file storage | Implement automatic cleanup of expired cache files |

## Security Considerations

1. **Never expose rate limit bypass mechanisms** - Keep admin overrides secure
2. **Use cryptographic identifiers** - Hash API keys and identifiers
3. **Implement IP-based fallback** - Use client IP when API key is unavailable
4. **Log rate limit violations** - Track excessive requests for security analysis
5. **Consider GeoIP blocking** - Block known malicious regions at rate limiter level

## Testing Checklist

- [ ] Test rate limiting with valid requests within limit
- [ ] Test rate limiting when limit is exceeded
- [ ] Verify X-RateLimit-* headers are present
- [ ] Verify Retry-After header on 429 response
- [ ] Test with multiple concurrent requests
- [ ] Test with Redis storage (if using distributed setup)
- [ ] Test with file-based storage
- [ ] Test rate limit reset functionality
- [ ] Test rate limiter with invalid/missing identifiers
- [ ] Test cleanup of expired cache files

## Reference Links

- [WHMCS Hook System](https://developers.whmcs.com/hooks/)
- [PHP Rate Limiting Patterns](https://php.net/manual/en/function.sleep.php)
- [Redis Rate Limiting](https://redis.io/commands/incr/)
- [RFC 6585 - Additional HTTP Status Codes](https://tools.ietf.org/html/rfc6585)
