# WHMCS Module Code Style Guide

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-coding-standards`, `module-packaging-guide`, `whmcs-security-standards`

---

## Overview

This guide establishes coding standards and best practices for WHMCS module development. Following these guidelines ensures code quality, security, maintainability, and compatibility with WHMCS standards.

---

## General Principles

### Core Standards

1. **PSR-12 Compliance** - Follow PHP-FIG PSR-12 coding standard
2. **Security First** - Always validate and sanitize input
3. **Error Handling** - Never expose sensitive information
4. **Documentation** - Document all functions and classes
5. **Testing** - Write testable, test-covered code

### File Organization

```
modules/
└── {type}/
    └── {module_name}/
        ├── {module_name}.php    # Main module file
        ├── class/
        │   └── Module.php       # Class implementation
        ├── src/
        │   ├── Api/
        │   ├── Models/
        │   └── Services/
        ├── templates/          # Admin/client templates
        ├── assets/
        │   ├── css/
        │   └── js/
        └── lang/               # Translation files
```

---

## Naming Conventions

### Classes and Files

```php
<?php
// Good: Descriptive, PascalCase
class StripePaymentGateway { }
class HostingServiceManager { }
class DomainRegistrationApi { }

// Bad: Abbreviated, unclear
class SPG { }
class HSM { }
class DRA { }

// File naming
// Class: modules/addons/example/addon/ExampleModule.php
// Namespace: WHMCS\Module\Addon\Example

// Class naming
namespace WHMCS\Module\Addon\Example;

class ExampleModule
{
    // ...
}
```

### Methods and Functions

```php
<?php
// Good: camelCase, descriptive verbs
public function createAccount(array $params): string
public function validateApiKey(string $key): bool
public function getInvoiceStatus(int $invoiceId): string

// Bad: abbreviations, unclear purpose
public function crAcct(array $p)
public function valKey($k)
public function getInvStat($id)

// Constants
const VERSION = '1.0.0';
const API_TIMEOUT = 30;
const MAX_RETRY_ATTEMPTS = 3;

// Private properties
private ApiClient $api;
private array $config;
private string $moduleName;
```

### Variables

```php
<?php
// Good: descriptive names
$clientId = $params['userid'];
$serviceId = $params['serviceid'];
$invoiceTotal = $invoice->total;

// Bad: single letters or unclear
$cid = $params['userid'];
$id = $params['serviceid'];
$t = $invoice->total;

// Arrays: plural nouns or descriptive suffixes
$serviceIds = [1, 2, 3];
$productConfigurations = [];
$gatewayResponse = [];

// Booleans: is/has/can prefixes
$isActive = true;
$hasPermission = false;
$canProcess = true;
```

---

## Documentation Standards

### PHPDoc Block

```php
<?php
/**
 * Create a new service account.
 *
 * This function provisions a new hosting account on the remote server
 * and initializes all required services.
 *
 * @param array $params Service and server parameters including:
 *                       - server: Server connection details
 *                       - serviceid: WHMCS service ID
 *                       - userid: Client ID
 *                       - username: Service username
 *                       - password: Service password
 *                       - domain: Service domain
 *
 * @return string 'success' on success, error message on failure
 *
 * @throws \Exception When API connection fails
 * @throws \InvalidArgumentException When required parameters missing
 *
 * @since 1.0
 * @example
 * ```php
 * $result = createAccount([
 *     'server' => $server,
 *     'serviceid' => 123,
 *     'domain' => 'example.com',
 * ]);
 * ```
 */
function createAccount(array $params): string
{
    // Implementation
}
```

### Inline Comments

```php
<?php
// Good: explain WHY, not WHAT
// Retry up to 3 times for transient API errors
for ($attempt = 1; $attempt <= 3; $attempt++) {
    try {
        $result = $api->create($params);
        break;
    } catch (TransientException $e) {
        if ($attempt === 3) {
            throw $e;
        }
        sleep(pow(2, $attempt)); // Exponential backoff
    }
}

// Bad: restating obvious code
// Increment counter
$counter++;

// Create new user
$user = new User();
```

---

## Security Standards

### Input Validation

```php
<?php
/**
 * Validate and sanitize all input
 */

// Integer validation
$serviceId = filter_input(INPUT_POST, 'service_id', FILTER_VALIDATE_INT);
if ($serviceId === false || $serviceId === null) {
    throw new \InvalidArgumentException('Invalid service ID');
}

// String sanitization
$domain = filter_input(INPUT_POST, 'domain', FILTER_SANITIZE_STRING);
$domain = preg_replace('/[^a-zA-Z0-9.-]/', '', $domain);

// Email validation
$email = filter_var($_POST['email'], FILTER_VALIDATE_EMAIL);
if ($email === false) {
    throw new \InvalidArgumentException('Invalid email address');
}

// URL validation
$url = filter_var($_POST['webhook_url'], FILTER_VALIDATE_URL);
if ($url === false) {
    throw new \InvalidArgumentException('Invalid URL');
}
```

### CSRF Protection

```php
<?php
// Required in all admin forms
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    check_token('WHMCS.admin.default');
}

// Client area forms
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    check_token('WHMCS.default');
}

// Generate form token in templates
// <input type="hidden" name="token" value="{$token}">
```

### SQL Injection Prevention

```php
<?php
// Use prepared statements
$stmt = Capsule::connection()->getPdo()
    ->prepare('SELECT * FROM mod_example WHERE id = ?');
$stmt->execute([$serviceId]);

// Capsule query builder
$results = Capsule::table('mod_example')
    ->where('user_id', $userId)
    ->whereIn('status', ['active', 'pending'])
    ->get();

// Avoid string concatenation in queries
// BAD: "SELECT * FROM tbl WHERE id = " . $id
// GOOD: "SELECT * FROM tbl WHERE id = ?", [$id]
```

### Password Handling

```php
<?php
// Never store plain passwords
// Never log passwords
// Use password_hash/password_verify

// Hash password
$hashedPassword = password_hash($plainPassword, PASSWORD_ARGON2ID);

// Verify password
if (password_verify($plainPassword, $hashedPassword)) {
    // Password correct
}

// Check for rehash needed
if (password_needs_rehash($hashedPassword, PASSWORD_ARGON2ID)) {
    $hashedPassword = password_hash($plainPassword, PASSWORD_ARGON2ID);
    // Update stored hash
}
```

---

## Error Handling

### Module Function Returns

```php
<?php
/**
 * Provisioning module: return 'success' or error string
 */
function mymodule_CreateAccount(array $params): string
{
    try {
        // Validate required parameters
        if (empty($params['username']) || empty($params['password'])) {
            return 'Error: Username and password are required';
        }

        // Perform action
        $api = new ApiClient($params['server']);
        $result = $api->createAccount($params);

        if (!$result['success']) {
            return 'Error: ' . ($result['error'] ?? 'Unknown error');
        }

        return 'success';

    } catch (\Exception $e) {
        logModuleCall(
            'MyModule',
            'CreateAccount',
            $params,
            ['error' => $e->getMessage()]
        );

        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Registrar module: return array
 */
function mymodule_RegisterDomain(array $params): array
{
    try {
        // Implementation
        $result = $api->registerDomain($params);

        return ['success' => true];

    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}
```

### Exception Handling

```php
<?php
/**
 * Custom exception classes
 */
namespace MyVendor\MyModule;

class ModuleException extends \Exception
{
    private array $context;

    public function __construct(string $message, array $context = [])
    {
        parent::__construct($message);
        $this->context = $context;
    }

    public function getContext(): array
    {
        return $this->context;
    }
}

class ApiException extends ModuleException
{
    private int $httpCode;

    public function __construct(string $message, int $httpCode = 0, array $context = [])
    {
        parent::__construct($message, $context);
        $this->httpCode = $httpCode;
    }

    public function getHttpCode(): int
    {
        return $this->httpCode;
    }
}

// Usage
throw new ApiException(
    'API request failed',
    500,
    ['endpoint' => $endpoint, 'method' => 'POST']
);
```

---

## Logging Standards

### Module Logging

```php
<?php
/**
 * Log all module operations
 */

// Basic activity logging
logActivity("MyModule: Processing order #{$orderId}");

// Detailed module logging (redacts sensitive data)
logModuleCall(
    'MyModule',           // Module name
    'CreateAccount',      // Function/action
    $params,              // Input parameters
    $response,            // Response/output
    [$params['password'], $params['serverpassword']] // Fields to redact
);
```

### Structured Logging

```php
<?php
/**
 * Structured log format
 */

$logEntry = [
    'timestamp' => date('Y-m-d H:i:s'),
    'module' => 'MyModule',
    'action' => 'process_order',
    'order_id' => $orderId,
    'client_id' => $clientId,
    'amount' => $amount,
    'status' => 'success',
    'duration_ms' => (microtime(true) - $startTime) * 1000,
];

logActivity(json_encode($logEntry));
```

---

## Code Structure

### Module File Structure

```php
<?php
/**
 * WHMCS Provisioning Module
 *
 * Module: My Cloud Server
 * Version: 1.0.0
 * Author: Your Name
 * Description: Cloud server provisioning module
 */

// Prevent direct access
if (!defined('WHMCS')) {
    die('This file cannot be accessed directly');
}

// ============================================
// METADATA
// ============================================

/**
 * Module metadata
 *
 * @return array Module information
 */
function mycloudsvr_MetaData(): array
{
    return [
        'DisplayName' => 'My Cloud Server',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
        'NonSSLPort' => 443,
    ];
}

// ============================================
// CONFIGURATION
// ============================================

/**
 * Module configuration options
 *
 * @param array $params Configuration parameters
 * @return array Configuration fields
 */
function mycloudsvr_ConfigOptions(array $params): array
{
    return [
        'plan' => [
            'Type' => 'dropdown',
            'Options' => 'starter,business,enterprise',
            'Default' => 'starter',
            'Description' => 'Select the hosting plan',
        ],
    ];
}

// ============================================
// LIFECYCLE FUNCTIONS
// ============================================

/**
 * Create new account
 *
 * @param array $params Service parameters
 * @return string 'success' or error message
 */
function mycloudsvr_CreateAccount(array $params): string
{
    // Implementation
}

/**
 * Suspend account
 *
 * @param array $params Service parameters
 * @return string 'success' or error message
 */
function mycloudsvr_SuspendAccount(array $params): string
{
    // Implementation
}

/**
 * Terminate account
 *
 * @param array $params Service parameters
 * @return string 'success' or error message
 */
function mycloudsvr_TerminateAccount(array $params): string
{
    // Implementation
}

// ============================================
// UTILITY FUNCTIONS
// ============================================

/**
 * Build API client
 *
 * @param array $server Server parameters
 * @return ApiClient
 */
function buildApiClient(array $server): ApiClient
{
    return new ApiClient([
        'api_key' => $server['configoption1'],
        'endpoint' => $server['serverhostname'],
    ]);
}
```

---

## Testing Standards

### Unit Testing

```php
<?php
/**
 * PHPUnit test example
 */

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;
use WHMCS\Module\Server\MyCloudServer\ApiClient;

class ApiClientTest extends TestCase
{
    private ApiClient $client;

    protected function setUp(): void
    {
        $this->client = new ApiClient([
            'api_key' => 'test_key',
            'endpoint' => 'https://api.test.com',
        ]);
    }

    public function testPingReturnsSuccess(): void
    {
        $response = $this->client->ping();

        $this->assertTrue($response['success']);
    }

    public function testCreateServerWithValidParams(): void
    {
        $params = [
            'hostname' => 'test.example.com',
            'plan' => 'starter',
        ];

        $result = $this->client->createServer($params);

        $this->assertTrue($result['success']);
        $this->assertArrayHasKey('server_id', $result);
    }

    public function testCreateServerWithMissingParamsThrows(): void
    {
        $this->expectException(\InvalidArgumentException::class);

        $this->client->createServer([]);
    }
}
```

---

## Related Documentation

- [Code Style Guide](code-style-guide.md)
- [Module API Reference](whmcs-module-api.md)
- [Security Best Practices](security-best-practices.md)
- [Module Error Handling Guide](module-error-handling-guide.md)
- [Module Testing Strategies](module-testing-strategies.md)
