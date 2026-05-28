# WHMCS Code Style Guide
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Coding standards for WHMCS module development.

## PHP Standards

### Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Classes | PascalCase | `ApiClient`, `WebhookHandler` |
| Methods | camelCase | `createAccount()`, `getStats()` |
| Properties | camelCase | `private $apiKey` |
| Constants | SCREAMING_SNAKE | `MAX_RETRY_COUNT` |
| Variables | snake_case | `$service_id`, `$api_key` |
| Functions | snake_case | `process_payment()` |

### Functions

```php
// WHMCS module functions: {module}_FunctionName
function {module}_CreateAccount(array $params): string { }
function {module}_SuspendAccount(array $params): string { }

// Private helper functions
function format_currency(float $amount): string { }
function sanitize_input(string $input): string { }
```

### Arrays

```php
// Preferred syntax
$config = [
    'name' => 'value',
    'enabled' => true,
];

// Align values for readability
$config = [
    'name'         => 'Module Name',
    'description' => 'Description',
    'enabled'     => true,
];
```

### Control Structures

```php
// Use braces for all
if ($condition) {
    // code
} elseif ($other) {
    // code
} else {
    // code
}

// Switch with comments
switch ($status) {
    case 'active':
        // Handle active
        break;

    case 'suspended':
        // Handle suspended
        break;
}
```

## Code Examples

### Good
```php
function {module}_CreateAccount(array $params): string {
    try {
        $api = new ApiClient($params);
        $result = $api->createInstance([
            'domain' => $params['domain'],
            'plan' => $params['configoption1'],
        ]);

        saveCustomFieldValue($params['serviceid'], 'instance_id', $result['id']);

        return 'success';
    } catch (\Exception $e) {
        logActivity('{Module} Error: ' . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}
```

### Bad
```php
function {module}_CreateAccount($params) {
    $api=new ApiClient($params);
    $result=$api->createInstance(array(
        'domain'=>$params['domain'],
        'plan'=>$params['configoption1']
    ));
    saveCustomFieldValue($params['serviceid'],'instance_id',$result['id']);
    return 'success';
}
```

## Documentation

```php
/**
 * Create a new account instance.
 *
 * @param array $params WHMCS parameters including:
 *                      - serviceid: Service ID
 *                      - domain: Domain name
 *                      - configoption1: Selected plan
 *                      - serverusername: API key
 *
 * @return string 'success' on success, error message otherwise
 */
function {module}_CreateAccount(array $params): string { }
```

---

**Related Skills:**
- whmcs-testing-qa
- whmcs-deployment
