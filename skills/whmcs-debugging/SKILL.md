# WHMCS Module Debugging Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for debugging WHMCS modules effectively.

## When to Use

- Finding module errors
- Troubleshooting API issues
- Debugging database operations

## Debugging Techniques

### Logging Debug Info
```php
// Detailed logging
logActivity('{Module} Debug: ' . print_r($data, true));
logActivity('{Module} Request: ' . json_encode($_REQUEST));
```

### Xdebug Configuration
```ini
# php.ini
[xdebug]
xdebug.mode=debug
xdebug.start_with_request=yes
xdebug.client_host=127.0.0.1
xdebug.client_port=9000
```

### WHMCS Debug Mode
```php
// Enable in configuration.php
$display_errors = true;

// Or via hook
add_hook('AfterInitialConfig', 1, function() {
    ini_set('display_errors', 1);
    error_reporting(E_ALL);
});
```

### Database Query Debugging
```php
// Log all queries
Capsule::connection()->enableQueryLog();
$queries = Capsule::getQueryLog();

foreach ($queries as $query) {
    logActivity('Query: ' . $query['query']);
}
```

### API Debugging
```php
// Log API requests and responses
private function debugRequest(string $method, string $url, array $data): array {
    logActivity("API Request: $method $url");
    logActivity('Data: ' . json_encode($data));

    $result = $this->request($method, $url, $data);

    logActivity('API Response: ' . json_encode($result));
    return $result;
}
```

### Template Debugging
```smarty
{debug}
{$variable|var_dump}
```

---

**Related Skills:**
- whmcs-testing-qa
- whmcs-logging
- whmcs-error-handling
