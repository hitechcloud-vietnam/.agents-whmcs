# WHMCS Middleware

Complete guide to middleware patterns.

## Overview

Implement middleware for request/response processing.

## Middleware Interface

```php
<?php
/**
 * Middleware interface
 */
interface Middleware
{
    /**
     * Handle request
     */
    public function handle(array $request, callable $next): array;
}

/**
 * Authentication middleware
 */
class AuthMiddleware implements Middleware
{
    public function handle(array $request, callable $next): array
    {
        if (!isset($request['headers']['Authorization'])) {
            return [
                'status' => 401,
                'body' => ['error' => 'Unauthorized'],
            ];
        }
        
        return $next($request);
    }
}

/**
 * Logging middleware
 */
class LogMiddleware implements Middleware
{
    public function handle(array $request, callable $next): array
    {
        $start = microtime(true);
        
        $response = $next($request);
        
        $duration = (microtime(true) - $start) * 1000;
        
        logActivity("Request: {$request['method']} {$request['path']} - {$duration}ms");
        
        return $response;
    }
}
```

## Best Practices

1. **Single purpose** - Each middleware does one thing
2. **Order matters** - Register in correct order
3. **Error handling** - Handle failures gracefully
4. **Performance** - Don't add unnecessary overhead
5. **Reusability** - Design for reuse
6. **Testing** - Test middleware independently

## Related Documentation

- [whmcs-advanced-api.md](whmcs-advanced-api.md)
- [whmcs-advanced-security.md](whmcs-advanced-security.md)
