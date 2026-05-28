# WHMCS REST API Builder Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building REST APIs within WHMCS modules.

## When to Use

- Creating module APIs
- Building webhook endpoints
- Exposing data to external systems

## REST API Patterns

```php
<?php
// modules/addons/{module}/api.php
if (!defined("WHMCS")) { die("Direct access denied"); }

header('Content-Type: application/json');
header('Access-Control-Allow-Origin: *');
header('Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS');

if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
    exit(0);
}

// Route handling
$path = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
$method = $_SERVER['REQUEST_METHOD'];

try {
    $result = match ($path) {
        '/api/services' => handleGetServices($_GET),
        '/api/services/{id}' => handleGetService($path),
        '/api/services' => handleCreateService($_POST),
        default => throw new \Exception('Not Found', 404),
    };

    http_response_code(200);
    echo json_encode(['success' => true, 'data' => $result]);

} catch (\Exception $e) {
    http_response_code($e->getCode() ?: 500);
    echo json_encode(['success' => false, 'error' => $e->getMessage()]);
}

function handleGetServices(array $params): array {
    $query = Capsule::table('tblhosting');

    if (!empty($params['status'])) {
        $query->where('domainstatus', $params['status']);
    }

    return $query->limit(50)->get();
}

function handleGetService(string $path): array {
    preg_match('/\/api\/services\/(\d+)/', $path, $matches);
    $serviceId = (int) $matches[1];

    $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();

    if (!$service) {
        throw new \Exception('Service not found', 404);
    }

    return $service;
}

function handleCreateService(array $data): array {
    // Validate required fields
    if (empty($data['client_id']) || empty($data['product_id'])) {
        throw new \Exception('Missing required fields', 400);
    }

    $result = localAPI('CreateService', [
        'clientid' => $data['client_id'],
        'pid' => $data['product_id'],
    ]);

    return $result;
}
```

---

**Related Skills:**
- whmcs-api-integration
- whmcs-webhook-handler
- whmcs-security-hardening
