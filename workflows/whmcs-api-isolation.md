# WHMCS API Isolation Workflow

## Overview
Configure API rate limiting and isolation per client/tenant.

## Prerequisites
- WHMCS v8.0+
- API access

## Step-by-Step Guide

### Step 1: Rate Limiting Middleware
```php
<?php
class RateLimitMiddleware
{
    protected array $limits = [
        'default' => ['requests' => 60, 'period' => 60],
        'premium' => ['requests' => 600, 'period' => 60],
        'enterprise' => ['requests' => 6000, 'period' => 60],
    ];

    public function checkLimit(string $clientId): bool
    {
        $tier = $this->getClientTier($clientId);
        $limit = $this->limits[$tier] ?? $this->limits['default'];
        
        $key = "rate_limit:$clientId";
        $current = (int) \Cache::get($key, 0);
        
        if ($current >= $limit['requests']) {
            return false;
        }
        
        \Cache::put($key, $current + 1, $limit['period']);
        return true;
    }
}
```

### Step 2: API Authentication
```php
<?php
function validate_api_key(string $apiKey): ?array
{
    $key = \WHMCS\Database\Capsule::table('mod_api_keys')
        ->where('api_key', hash('sha256', $apiKey))
        ->where('active', true)
        ->first();
    
    return $key ?: null;
}
```

## Checklist
- Rate limits configured
- Per-client limits enforced
- API authentication implemented
- Monitoring enabled
