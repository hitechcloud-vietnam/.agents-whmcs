# WHMCS Module Class Autoloading

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-module-api`, `whmcs-autoloader`, `dependency-injection`

---

## Overview

WHMCS uses PSR-4 autoloading standards for loading module classes. This guide covers autoloading configuration, namespace conventions, and patterns for organizing module code.

---

## PSR-4 Autoloading Standard

### What is PSR-4?

PSR-4 defines a standard for autoloading classes from file paths. It maps namespaces to directory structures, allowing automatic class loading without manual include statements.

### Basic Principle

```
Namespace: WHMCS\Module\Addon\Example
Path: modules/addons/example/src/Example.php
```

---

## Module Namespace Structure

### Standard Module Namespaces

```
modules/
├── addons/
│   └── example/
│       ├── example.php              # Module entry point
│       └── src/
│           ├── ExampleModule.php     # WHMCS\Module\Addon\Example\ExampleModule
│           ├── Services/
│           │   └── ApiService.php    # WHMCS\Module\Addon\Example\Services\ApiService
│           └── Models/
│               └── ExampleModel.php  # WHMCS\Module\Addon\Example\Models\ExampleModel
│
├── servers/
│   └── myserver/
│       ├── myserver.php              # Module entry point
│       └── src/
│           ├── ServerModule.php      # WHMCS\Module\Server\MyServer\ServerModule
│           └── Api/
│               └── ApiClient.php     # WHMCS\Module\Server\MyServer\Api\ApiClient
│
└── gateways/
    └── mygateway/
        ├── mygateway.php             # Module entry point
        └── src/
            └── GatewayModule.php     # WHMCS\Module\Gateway\MyGateway\GatewayModule
```

---

## Composer Configuration

### composer.json Setup

```json
{
    "name": "vendor/module-name",
    "description": "Module description",
    "type": "whmcs-module",
    "license": "proprietary",
    "require": {
        "php": ">=7.4",
        "whmcs/core": "^8.0"
    },
    "autoload": {
        "psr-4": {
            "WHMCS\\Module\\Addon\\Example\\": "src/",
            "WHMCS\\Module\\Addon\\Example\\Services\\": "src/Services/",
            "WHMCS\\Module\\Addon\\Example\\Models\\": "src/Models/",
            "WHMCS\\Module\\Addon\\Example\\Contracts\\": "src/Contracts/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "WHMCS\\Module\\Addon\\Example\\Tests\\": "tests/"
        }
    }
}
```

### Module-Specific Autoloading

```php
<?php
/**
 * Custom autoloader for module (fallback if composer not used)
 */

spl_autoload_register(function ($class) {
    // Only handle our module namespace
    $prefix = 'WHMCS\\Module\\Addon\\Example\\';

    $len = strlen($prefix);
    if (strncmp($prefix, $class, $len) !== 0) {
        return;
    }

    // Get relative class name
    $relativeClass = substr($class, $len);

    // Replace namespace separators with directory separators
    $file = __DIR__ . '/src/' . str_replace('\\', '/', $relativeClass) . '.php';

    // Load the file
    if (file_exists($file)) {
        require $file;
    }
});
```

---

## Module Class Structure

### Addon Module Class

```php
<?php
// modules/addons/example/src/ExampleModule.php

namespace WHMCS\Module\Addon\Example;

use WHMCS\Module\Addon\Example\Services\ApiService;
use WHMCS\Module\Addon\Example\Models\DataModel;
use WHMCS\Module\Addon\Example\Contracts\HookInterface;

class ExampleModule
{
    private ApiService $api;
    private DataModel $model;
    private array $config;

    public function __construct(array $config = [])
    {
        $this->config = $config;
        $this->api = new ApiService($config['api_key'] ?? '');
        $this->model = new DataModel();
    }

    /**
     * Module activation
     */
    public function activate(): array
    {
        // Create database tables
        $this->model->createTables();

        return [
            'status' => 'success',
            'description' => 'Module activated successfully',
        ];
    }

    /**
     * Module deactivation
     */
    public function deactivate(): array
    {
        $this->model->dropTables();

        return [
            'status' => 'success',
            'description' => 'Module deactivated',
        ];
    }

    /**
     * Render admin output
     */
    public function output(array $vars): string
    {
        $data = $this->model->getStatistics();

        return $this->renderTemplate('admin/dashboard', [
            'statistics' => $data,
            'version' => self::VERSION,
        ]);
    }

    /**
     * Render client area
     */
    public function clientArea(array $vars): array
    {
        $userId = $_SESSION['uid'];

        return [
            'pagetitle' => 'My Module',
            'templatefile' => 'clientarea/index',
            'vars' => [
                'userData' => $this->model->getUserData($userId),
            ],
        ];
    }
}
```

### Service Classes

```php
<?php
// modules/addons/example/src/Services/ApiService.php

namespace WHMCS\Module\Addon\Example\Services;

use WHMCS\Module\Addon\Example\Contracts\ApiClientInterface;
use WHMCS\Module\Addon\Example\Exceptions\ApiException;

class ApiService implements ApiClientInterface
{
    private string $apiKey;
    private string $baseUrl;
    private int $timeout;

    public function __construct(string $apiKey, string $baseUrl = '', int $timeout = 30)
    {
        $this->apiKey = $apiKey;
        $this->baseUrl = $baseUrl ?: 'https://api.example.com';
        $this->timeout = $timeout;
    }

    /**
     * Make API request
     */
    public function request(string $method, string $endpoint, array $data = []): array
    {
        $url = rtrim($this->baseUrl, '/') . '/' . ltrim($endpoint, '/');

        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
        ]);

        if ($method !== 'GET') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new ApiException("cURL error: {$error}");
        }

        $decoded = json_decode($response, true);

        if ($httpCode >= 400) {
            throw new ApiException(
                $decoded['message'] ?? 'API error',
                $httpCode
            );
        }

        return $decoded;
    }

    public function get(string $endpoint): array
    {
        return $this->request('GET', $endpoint);
    }

    public function post(string $endpoint, array $data): array
    {
        return $this->request('POST', $endpoint, $data);
    }
}
```

### Model Classes

```php
<?php
// modules/addons/example/src/Models/DataModel.php

namespace WHMCS\Module\Addon\Example\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Schema\Blueprint;

class DataModel
{
    private string $tableName = 'mod_example_data';

    /**
     * Create module tables
     */
    public function createTables(): void
    {
        if (!Capsule::schema()->hasTable($this->tableName)) {
            Capsule::schema()->create($this->tableName, function (Blueprint $t) {
                $t->increments('id');
                $t->unsignedInteger('user_id');
                $t->string('external_id')->nullable();
                $t->json('data')->nullable();
                $t->timestamps();

                $t->index('user_id');
                $t->index('external_id');
            });
        }
    }

    /**
     * Get user data
     */
    public function getUserData(int $userId): ?array
    {
        $record = Capsule::table($this->tableName)
            ->where('user_id', $userId)
            ->first();

        return $record ? (array) $record : null;
    }

    /**
     * Save user data
     */
    public function saveUserData(int $userId, array $data): int
    {
        $existing = $this->getUserData($userId);

        if ($existing) {
            Capsule::table($this->tableName)
                ->where('user_id', $userId)
                ->update([
                    'data' => json_encode($data),
                    'updated_at' => date('Y-m-d H:i:s'),
                ]);

            return $existing['id'];
        }

        return Capsule::table($this->tableName)
            ->insertGetId([
                'user_id' => $userId,
                'data' => json_encode($data),
                'created_at' => date('Y-m-d H:i:s'),
                'updated_at' => date('Y-m-d H:i:s'),
            ]);
    }
}
```

---

## Dependency Injection

### Service Container Pattern

```php
<?php
// modules/addons/example/src/Container.php

namespace WHMCS\Module\Addon\Example;

use WHMCS\Module\Addon\Example\Services\ApiService;
use WHMCS\Module\Addon\Example\Models\DataModel;

class Container
{
    private static array $services = [];

    public static function register(string $name, $service): void
    {
        self::$services[$name] = $service;
    }

    public static function get(string $name)
    {
        if (!isset(self::$services[$name])) {
            self::$services[$name] = self::resolve($name);
        }

        return self::$services[$name];
    }

    private static function resolve(string $name)
    {
        return match ($name) {
            'api' => new ApiService(
                Capsule::table('tbladdonmodules')
                    ->where('module', 'example')
                    ->value('setting') ?? ''
            ),
            'model' => new DataModel(),
            default => throw new \Exception("Unknown service: {$name}"),
        };
    }

    public static function reset(): void
    {
        self::$services = [];
    }
}
```

### Using the Container

```php
<?php
// modules/addons/example/example.php

use WHMCS\Module\Addon\Example\Container;

if (!defined('WHMCS')) {
    die('This file cannot be accessed directly');
}

/**
 * Module entry point
 */
function example_config(): array
{
    return [
        'name' => 'Example Module',
        'description' => 'Example module with autoloading',
        'version' => '1.0.0',
    ];
}

function example_activate(): array
{
    return Container::get('model')->createTables()
        ? ['status' => 'success']
        : ['status' => 'error'];
}

function example_output(array $vars): void
{
    $module = new ExampleModule(Container::get('api'));
    echo $module->output($vars);
}
```

---

## Contract/Interface Definitions

```php
<?php
// modules/addons/example/src/Contracts/ApiClientInterface.php

namespace WHMCS\Module\Addon\Example\Contracts;

interface ApiClientInterface
{
    /**
     * Make API request
     */
    public function request(string $method, string $endpoint, array $data = []): array;

    /**
     * GET request
     */
    public function get(string $endpoint): array;

    /**
     * POST request
     */
    public function post(string $endpoint, array $data): array;
}
```

```php
<?php
// modules/addons/example/src/Contracts/HookInterface.php

namespace WHMCS\Module\Addon\Example\Contracts;

interface HookInterface
{
    /**
     * Get hook definitions
     *
     * @return array Array of hook definitions
     */
    public static function getHooks(): array;
}
```

---

## Autoloading in Hooks

### Registering Hooks via Classes

```php
<?php
// modules/addons/example/src/Hooks/ClientHooks.php

namespace WHMCS\Module\Addon\Example\Hooks;

use WHMCS\Module\Addon\Example\Container;

class ClientHooks
{
    /**
     * Get all hooks
     */
    public static function getHooks(): array
    {
        return [
            [
                'hook' => 'ClientAdd',
                'priority' => 1,
                'handler' => [self::class, 'onClientAdd'],
            ],
            [
                'hook' => 'ClientLogin',
                'priority' => 1,
                'handler' => [self::class, 'onClientLogin'],
            ],
        ];
    }

    /**
     * Handle new client registration
     */
    public static function onClientAdd(array $vars): void
    {
        $model = Container::get('model');

        // Sync to external service
        $model->saveUserData($vars['userid'], [
            'email' => $vars['email'],
            'name' => $vars['firstname'] . ' ' . $vars['lastname'],
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    /**
     * Handle client login
     */
    public static function onClientLogin(array $vars): void
    {
        logActivity("Client logged in: " . $vars['userid']);
    }
}
```

```php
<?php
// modules/addons/example/hooks.php

use WHMCS\Module\Addon\Example\Hooks\ClientHooks;

// Register hooks
foreach (ClientHooks::getHooks() as $hook) {
    add_hook(
        $hook['hook'],
        $hook['priority'],
        $hook['handler']
    );
}
```

---

## Best Practices

1. **Use namespaces** - All classes should be in proper namespaces
2. **Follow PSR-4** - Map namespaces to directory structure
3. **Single Responsibility** - Each class has one purpose
4. **Dependency Injection** - Inject dependencies rather than instantiating
5. **Use Contracts** - Define interfaces for services
6. **Composer for large modules** - Use composer.json for complex autoloading
7. **Fallback autoloader** - Provide fallback for simple modules

---

## Related Documentation

- [Dependency Injection](dependency-injection.md)
- [Module API Reference](whmcs-module-api.md)
- [Service Layer](service-layer.md)
- [Builder Pattern](builder-pattern.md)
