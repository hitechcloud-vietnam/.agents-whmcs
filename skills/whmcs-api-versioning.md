# WHMCS API Versioning

## Skill Description
Implement API versioning strategy for WHMCS modules to maintain backward compatibility, support multiple API versions, and manage smooth migrations between versions.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+
- Understanding of REST API design
- Basic routing knowledge

## Step-by-Step Implementation

### 1. Version Manager
```php
<?php
// includes/versioning/ApiVersionManager.php

namespace WHMCS\Module\YourModule\Versioning;

class ApiVersionManager
{
    private const CURRENT_VERSION = 'v2';
    private const SUPPORTED_VERSIONS = ['v1', 'v2'];
    private const DEPRECATED_VERSIONS = ['v0'];

    private string $requestedVersion;
    private array $versionConfig = [];

    public function __construct(?string $requestedVersion = null)
    {
        $this->requestedVersion = $requestedVersion ?? $this->detectVersion();
        $this->loadVersionConfig();
    }

    private function detectVersion(): string
    {
        // Check URL path for version
        $path = parse_url($_SERVER['REQUEST_URI'] ?? '', PHP_URL_PATH);

        if (preg_match('/\/api\/([^\/]+)\//', $path, $matches)) {
            return $matches[1];
        }

        // Check Accept header
        $accept = $_SERVER['HTTP_ACCEPT'] ?? '';
        if (preg_match('/application\/vnd\.yourmodule\.([^\s;]+)/', $accept, $matches)) {
            return $matches[1];
        }

        // Check X-API-Version header
        $version = $_SERVER['HTTP_X_API_VERSION'] ?? '';
        if (in_array($version, self::SUPPORTED_VERSIONS)) {
            return $version;
        }

        return self::CURRENT_VERSION;
    }

    private function loadVersionConfig(): void
    {
        $this->versionConfig = [
            'v1' => [
                'deprecated' => true,
                'sunset_date' => '2025-12-31',
                'breaking_changes' => [
                    'UserController@show' => 'Response format changed',
                    'InvoiceController@create' => 'Required fields modified'
                ],
                'transformers' => [
                    'user' => V1UserTransformer::class,
                    'invoice' => V1InvoiceTransformer::class
                ]
            ],
            'v2' => [
                'deprecated' => false,
                'sunset_date' => null,
                'breaking_changes' => [],
                'transformers' => [
                    'user' => V2UserTransformer::class,
                    'invoice' => V2InvoiceTransformer::class
                ]
            ]
        ];
    }

    public function getVersion(): string
    {
        return $this->requestedVersion;
    }

    public function isSupported(): bool
    {
        return in_array($this->requestedVersion, self::SUPPORTED_VERSIONS);
    }

    public function isDeprecated(): bool
    {
        return in_array($this->requestedVersion, self::DEPRECATED_VERSIONS)
            || ($this->versionConfig[$this->requestedVersion]['deprecated'] ?? false);
    }

    public function getSunsetDate(): ?string
    {
        return $this->versionConfig[$this->requestedVersion]['sunset_date'] ?? null;
    }

    public function getDeprecationHeaders(): array
    {
        $headers = [];

        if ($this->isDeprecated()) {
            $headers['Deprecation'] = 'true';
            $headers['Sunset'] = $this->getSunsetDate();
            $headers['Link'] = sprintf(
                '<%s>; rel="deprecation"; type="text/html"',
                $this->getMigrationGuideUrl()
            );
        }

        return $headers;
    }

    public function getMigrationGuideUrl(): string
    {
        return 'https://docs.yourmodule.com/migration/' . $this->requestedVersion;
    }

    public function getController(string $controller): ?string
    {
        $versionedController = sprintf(
            'WHMCS\\Module\\YourModule\\Api\\Controllers\\%s\\%sController',
            ucfirst($this->requestedVersion),
            $controller
        );

        if (class_exists($versionedController)) {
            return $versionedController;
        }

        // Fall back to current version
        $currentController = sprintf(
            'WHMCS\\Module\\YourModule\\Api\\Controllers\\%s\\%sController',
            ucfirst(self::CURRENT_VERSION),
            $controller
        );

        return class_exists($currentController) ? $currentController : null;
    }

    public function getTransformer(string $resource): ?string
    {
        $transformerClass = $this->versionConfig[$this->requestedVersion]['transformers'][$resource] ?? null;

        if ($transformerClass && class_exists($transformerClass)) {
            return $transformerClass;
        }

        return null;
    }
}
```

### 2. Versioned Router
```php
<?php
// includes/versioning/VersionedRouter.php

namespace WHMCS\Module\YourModule\Versioning;

use WHMCS\Module\YourModule\Api\ApiResponse;

class VersionedRouter
{
    private ApiVersionManager $versionManager;
    private array $routes = [];

    public function __construct()
    {
        $this->versionManager = new ApiVersionManager();
    }

    public function registerRoute(
        string $method,
        string $path,
        string $controller,
        string $action,
        ?array $middleware = null
    ): self {
        $this->routes[] = [
            'method' => strtoupper($method),
            'path' => $path,
            'controller' => $controller,
            'action' => $action,
            'middleware' => $middleware ?? []
        ];

        return $this;
    }

    public function dispatch(): void
    {
        // Check version support
        if (!$this->versionManager->isSupported()) {
            $this->sendVersionNotSupportedResponse();
            return;
        }

        // Send deprecation headers if needed
        $this->sendDeprecationHeaders();

        $method = $_SERVER['REQUEST_METHOD'];
        $path = parse_url($_SERVER['REQUEST_URI'] ?? '', PHP_URL_PATH);

        foreach ($this->routes as $route) {
            if ($route['method'] !== $method) {
                continue;
            }

            $params = $this->matchPath($route['path'], $path);

            if ($params !== false) {
                $this->executeRoute($route, $params);
                return;
            }
        }

        ApiResponse::error('Endpoint not found', 404, [], 'NOT_FOUND')->send();
    }

    private function matchPath(string $pattern, string $path): array|false
    {
        $regex = preg_replace('/\{([a-zA-Z_]+)\}/', '(?P<$1>[^/]+)', $pattern);
        $regex = '#^' . $regex . '$#';

        if (preg_match($regex, $path, $matches)) {
            return array_filter($matches, 'is_string', ARRAY_FILTER_USE_KEY);
        }

        return false;
    }

    private function executeRoute(array $route, array $params): void
    {
        $controllerClass = $this->versionManager->getController($route['controller']);

        if (!$controllerClass) {
            ApiResponse::serverError('Controller not found for version ' . $this->versionManager->getVersion())->send();
            return;
        }

        // Execute middleware
        foreach ($route['middleware'] as $middleware) {
            $result = $this->executeMiddleware($middleware);
            if ($result !== true) {
                return;
            }
        }

        $controller = new $controllerClass();
        $action = $route['action'];

        if (!method_exists($controller, $action)) {
            ApiResponse::serverError('Action not found: ' . $action)->send();
            return;
        }

        call_user_func_array([$controller, $action], $params);
    }

    private function executeMiddleware(string $middleware): bool
    {
        // Middleware implementation
        return true;
    }

    private function sendDeprecationHeaders(): void
    {
        $headers = $this->versionManager->getDeprecationHeaders();

        foreach ($headers as $name => $value) {
            header("$name: $value");
        }
    }

    private function sendVersionNotSupportedResponse(): void
    {
        http_response_code(400);
        header('Content-Type: application/json');

        echo json_encode([
            'success' => false,
            'error' => [
                'message' => 'API version not supported',
                'code' => 'VERSION_NOT_SUPPORTED',
                'supported_versions' => ApiVersionManager::SUPPORTED_VERSIONS,
                'current_version' => ApiVersionManager::CURRENT_VERSION
            ]
        ]);
    }
}
```

### 3. Response Transformers
```php
<?php
// includes/versioning/Transformers/V2UserTransformer.php

namespace WHMCS\Module\YourModule\Versioning\Transformers;

class V2UserTransformer
{
    public function transform(array $user): array
    {
        return [
            'id' => (int) $user['id'],
            'type' => 'user',
            'attributes' => [
                'first_name' => $user['firstname'],
                'last_name' => $user['lastname'],
                'email' => $user['email'],
                'company' => $user['companyname'] ?? null,
                'created_at' => $user['created_at'] ?? null,
                'updated_at' => $user['updated_at'] ?? null
            ],
            'relationships' => [
                'clients' => [
                    'data' => $this->transformClients($user['clients'] ?? [])
                ]
            ]
        ];
    }

    private function transformClients(array $clients): array
    {
        return array_map(function ($client) {
            return [
                'type' => 'client',
                'id' => (string) $client['id']
            ];
        }, $clients);
    }

    public function transformCollection(array $users): array
    {
        return [
            'data' => array_map([$this, 'transform'], $users),
            'meta' => [
                'total' => count($users)
            ]
        ];
    }
}
```

```php
<?php
// includes/versioning/Transformers/V1UserTransformer.php

namespace WHMCS\Module\YourModule\Versioning\Transformers;

class V1UserTransformer
{
    public function transform(array $user): array
    {
        // Legacy format for v1
        return [
            'user_id' => (int) $user['id'],
            'firstname' => $user['firstname'],
            'lastname' => $user['lastname'],
            'email_address' => $user['email'],
            'company_name' => $user['companyname'] ?? '',
            'date_created' => $user['created_at'] ?? null
        ];
    }

    public function transformCollection(array $users): array
    {
        return [
            'users' => array_map([$this, 'transform'], $users),
            'count' => count($users)
        ];
    }
}
```

### 4. API Endpoint Example
```php
<?php
// api/index.php

require_once __DIR__ . '/../../init.php';

use WHMCS\Module\YourModule\Versioning\VersionedRouter;

$router = new VersionedRouter();

// Register routes
$router->registerRoute('GET', '/api/v2/users', 'User', 'index')
       ->registerRoute('GET', '/api/v2/users/{id}', 'User', 'show')
       ->registerRoute('POST', '/api/v2/users', 'User', 'store')
       ->registerRoute('PUT', '/api/v2/users/{id}', 'User', 'update')
       ->registerRoute('DELETE', '/api/v2/users/{id}', 'User', 'destroy');

$router->dispatch();
```

### 5. Version Migration Guide Generator
```php
<?php
// includes/versioning/MigrationGuideGenerator.php

namespace WHMCS\Module\YourModule\Versioning;

class MigrationGuideGenerator
{
    private array $changes = [
        'v1_to_v2' => [
            'endpoint_changes' => [
                [
                    'old' => '/api/v1/users',
                    'new' => '/api/v2/users',
                    'description' => 'Endpoint path updated to include version'
                ]
            ],
            'response_changes' => [
                'User' => [
                    'removed_fields' => ['date_created'],
                    'renamed_fields' => [
                        'email_address' => 'email',
                        'company_name' => 'company'
                    ],
                    'new_fields' => ['updated_at', 'relationships']
                ]
            ],
            'breaking_changes' => [
                'UserController::show now requires authentication',
                'Invoice items now required when creating invoice'
            ],
            'new_features' => [
                'Pagination support',
                'Filtering and sorting',
                'Expanded relationships'
            ]
        ]
    ];

    public function generate(string $fromVersion, string $toVersion): array
    {
        $key = $fromVersion . '_to_' . $toVersion;

        if (!isset($this->changes[$key])) {
            return ['error' => 'No migration path available'];
        }

        return $this->changes[$key];
    }

    public function generateMarkdown(string $fromVersion, string $toVersion): string
    {
        $migration = $this->generate($fromVersion, $toVersion);

        $md = "# Migration Guide: {$fromVersion} to {$toVersion}\n\n";

        if (isset($migration['endpoint_changes'])) {
            $md .= "## Endpoint Changes\n\n";
            foreach ($migration['endpoint_changes'] as $change) {
                $md .= "- `{$change['old']}` -> `{$change['new']}`\n";
                $md .= "  - {$change['description']}\n\n";
            }
        }

        if (isset($migration['breaking_changes'])) {
            $md .= "## Breaking Changes\n\n";
            foreach ($migration['breaking_changes'] as $change) {
                $md .= "- {$change}\n";
            }
            $md .= "\n";
        }

        if (isset($migration['new_features'])) {
            $md .= "## New Features\n\n";
            foreach ($migration['new_features'] as $feature) {
                $md .= "- {$feature}\n";
            }
        }

        return $md;
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Version detection conflicts | Use consistent version detection across all endpoints |
| Missing version-specific logic | Create transformer classes for each version |
| Broken backwards compatibility | Maintain separate controller directories per version |
| No deprecation timeline | Implement sunset dates with advance notice |
| Version negotiation failures | Provide clear error messages with supported versions |

## Security Considerations

1. **Maintain security across versions** - Don't reduce security in older versions
2. **Version-specific access controls** - Implement per-version permissions
3. **Audit logging per version** - Track API usage by version
4. **Rate limiting per version** - Apply limits consistently across versions
5. **Deprecation notices** - Inform users before removing versions

## Testing Checklist

- [ ] Test version detection from URL path
- [ ] Test version detection from Accept header
- [ ] Test version detection from X-API-Version header
- [ ] Test fallback to current version
- [ ] Test deprecation headers on deprecated versions
- [ ] Test 400 response for unsupported versions
- [ ] Test response transformation per version
- [ ] Test controller routing per version
- [ ] Test migration guide generation
- [ ] Test backwards compatibility

## Reference Links

- [API Versioning Best Practices](https://restfulapi.net/versioning/)
- [RFC 8594 - Sunset Header](https://tools.ietf.org/html/rfc8594)
- [Deprecation Header RFC](https://tools.ietf.org/html/draft-wilkinson-deprecation-header)
