# WHMCS Rate Limiting Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing rate limiting in WHMCS modules.

## When to Use

- API protection
- DoS prevention
- Usage quota management

## Rate Limiting Patterns

```php
<?php
class RateLimiter {
    private int $maxRequests;
    private int $windowSeconds;

    public function __construct(int $maxRequests = 100, int $windowSeconds = 60) {
        $this->maxRequests = $maxRequests;
        $this->windowSeconds = $windowSeconds;
    }

    public function check(string $identifier): bool {
        $key = 'rate_' . md5($identifier);
        $record = Capsule::table('mod_rate_limits')
            ->where('key', $key)
            ->where('window_start', '>', date('Y-m-d H:i:s', strtotime("-{$this->windowSeconds} seconds")))
            ->first();

        if (!$record) {
            Capsule::table('mod_rate_limits')->insert([
                'key' => $key,
                'request_count' => 1,
                'window_start' => date('Y-m-d H:i:s'),
            ]);
            return true;
        }

        if ($record->request_count >= $this->maxRequests) {
            return false;
        }

        Capsule::table('mod_rate_limits')
            ->where('key', $key)
            ->increment('request_count');

        return true;
    }

    public function getRemaining(string $identifier): int {
        $key = 'rate_' . md5($identifier);
        $record = Capsule::table('mod_rate_limits')
            ->where('key', $key)
            ->first();

        if (!$record) {
            return $this->maxRequests;
        }

        return max(0, $this->maxRequests - $record->request_count);
    }
}
```

---

**Related Skills:**
- whmcs-security-hardening
- whmcs-api-integration
