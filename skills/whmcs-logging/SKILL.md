# WHMCS Logging Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing comprehensive logging in WHMCS modules.

## When to Use

- Debugging module issues
- Tracking user actions
- Compliance logging

## Logging Patterns

### Log Table Setup

```php
function setupLogging(): void {
    Capsule::schema()->create('mod_{module}_logs', function($t) {
        $t->increments('id');
        $t->string('level', 20);        // debug, info, warning, error
        $t->string('category', 50);    // api, user, system
        $t->text('message');
        $t->json('context')->nullable();
        $t->string('ip_address', 45)->nullable();
        $t->integer('user_id')->unsigned()->nullable();
        $t->integer('admin_id')->unsigned()->nullable();
        $t->timestamp('created_at')->useCurrent();
        $t->index(['level', 'created_at']);
        $t->index(['user_id']);
        $t->index(['category']);
    });
}
```

### Logging Functions

```php
class ModuleLogger {
    const LEVEL_DEBUG = 'debug';
    const LEVEL_INFO = 'info';
    const LEVEL_WARNING = 'warning';
    const LEVEL_ERROR = 'error';

    public function log(string $level, string $message, array $context = []): void {
        $this->write([
            'level' => $level,
            'category' => $context['category'] ?? 'general',
            'message' => $message,
            'context' => $context,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null,
            'user_id' => session_get('uid'),
            'admin_id' => session_get('adminid'),
        ]);
    }

    public function debug(string $message, array $context = []): void {
        $this->log(self::LEVEL_DEBUG, $message, $context);
    }

    public function info(string $message, array $context = []): void {
        $this->log(self::LEVEL_INFO, $message, $context);
    }

    public function warning(string $message, array $context = []): void {
        $this->log(self::LEVEL_WARNING, $message, $context);
    }

    public function error(string $message, array $context = []): void {
        $this->log(self::LEVEL_ERROR, $message, $context);
    }

    private function write(array $data): void {
        Capsule::table('mod_{module}_logs')->insert($data);

        // Also write to WHMCS log
        if ($data['level'] === self::LEVEL_ERROR) {
            logActivity('[ERROR] ' . $data['message']);
        }
    }

    public function apiCall(string $endpoint, array $request, array $response, float $duration): void {
        $this->info('API call: ' . $endpoint, [
            'category' => 'api',
            'endpoint' => $endpoint,
            'duration_ms' => $duration,
            'request' => $this->sanitizeForLog($request),
            'response_code' => $response['code'] ?? 0,
        ]);
    }

    private function sanitizeForLog(array $data): array {
        // Remove sensitive data
        $sensitive = ['password', 'api_key', 'secret', 'token'];
        foreach ($sensitive as $key) {
            if (isset($data[$key])) {
                $data[$key] = '***REDACTED***';
            }
        }
        return $data;
    }
}
```

### Usage Examples

```php
// In module
$logger = new ModuleLogger();

$logger->info('Service created', [
    'service_id' => $serviceId,
    'user_id' => $userId,
]);

// API logging
$start = microtime(true);
$response = $api->request('POST', '/endpoint', $data);
$duration = (microtime(true) - $start) * 1000;

$logger->apiCall('/endpoint', $data, $response, $duration);

if ($response['error']) {
    $logger->error('API error', [
        'category' => 'api',
        'endpoint' => '/endpoint',
        'error' => $response['error'],
    ]);
}
```

### Log Viewer

```php
function viewLogs(array $filters): array {
    $query = Capsule::table('mod_{module}_logs');

    if (!empty($filters['level'])) {
        $query->where('level', $filters['level']);
    }

    if (!empty($filters['category'])) {
        $query->where('category', $filters['category']);
    }

    if (!empty($filters['from'])) {
        $query->where('created_at', '>=', $filters['from']);
    }

    if (!empty($filters['to'])) {
        $query->where('created_at', '<=', $filters['to']);
    }

    return $query
        ->orderBy('created_at', 'desc')
        ->limit(100)
        ->get()
        ->toArray();
}
```

## Checklist

- [ ] Log table with indexes
- [ ] Logger class
- [ ] Log levels
- [ ] Context tracking
- [ ] Sensitive data sanitization
- [ ] Log viewer

---

**Related Skills:**
- whmcs-error-handling
- whmcs-testing-qa
- whmcs-security-hardening