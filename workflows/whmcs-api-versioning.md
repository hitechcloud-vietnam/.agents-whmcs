# WHMCS API Versioning Workflow

## Purpose

Implement API versioning strategy for WHMCS custom API endpoints to ensure backward compatibility, smooth migrations, and clear deprecation paths. This workflow covers versioning patterns, implementation, and migration strategies.

## Prerequisites

- WHMCS v8.0+ installation
- Custom API endpoints implementation
- Version control system
- API documentation tools

## Workflow Steps

### Step 1: API Versioning Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                    API Versioning Strategy                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  Version Header                           │  │
│  │  Accept: application/vnd.whmcs.v1+json                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  URL Path Versioning                      │  │
│  │  /api/v1/clients  /api/v2/clients                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  Query Parameter Versioning              │  │
│  │  /api/clients?version=2                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ────────────────────────────────────────────────────────────── │
│                                                                  │
│  Version Lifecycle:                                             │
│                                                                  │
│  v1 (Current) ──────────▶ v2 (New) ──────────▶ v3 (Future)     │
│     │                      │                                   │
│     ▼                      ▼                                   │
│  [Deprecated]            [Current]                             │
│     │                      │                                   │
│     ▼                      ▼                                   │
│  [Sunset Date]          [Deprecated]                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: Versioned API Implementation

```php
<?php
// /var/www/html/whmcs/includes/api/VersionRouter.php

namespace WHMCS\API;

class VersionRouter
{
    private $versions = ['v1', 'v2', 'v3'];
    private $defaultVersion = 'v2';
    private $supportedFormats = ['json', 'xml'];
    
    /**
     * Route API request to appropriate version
     */
    public function route(): array
    {
        $version = $this->parseVersion();
        $action = $this->parseAction();
        $format = $this->parseFormat();
        
        // Validate version
        if (!in_array($version, $this->versions)) {
            return $this->error('Unsupported API version', 400, [
                'supported_versions' => $this->versions,
                'default_version' => $this->defaultVersion
            ]);
        }
        
        // Load versioned handler
        $handlerClass = "WHMCS\\API\\Versions\\{$version}\\{$action}";
        
        if (!class_exists($handlerClass)) {
            return $this->error('Unknown API action', 404, [
                'action' => $action,
                'version' => $version
            ]);
        }
        
        $handler = new $handlerClass();
        return $handler->handle($this->getRequestData());
    }
    
    /**
     * Parse version from request
     */
    private function parseVersion(): string
    {
        // Check header first
        $accept = $_SERVER['HTTP_ACCEPT'] ?? '';
        if (preg_match('/v(\d+)/', $accept, $matches)) {
            return 'v' . $matches[1];
        }
        
        // Check URL path
        $path = $_SERVER['REQUEST_URI'] ?? '';
        if (preg_match('/\/api\/v(\d+)\//', $path, $matches)) {
            return 'v' . $matches[1];
        }
        
        // Check query parameter
        return $_GET['version'] ?? $this->defaultVersion;
    }
    
    /**
     * Parse action from request
     */
    private function parseAction(): string
    {
        $path = $_SERVER['REQUEST_URI'] ?? '';
        preg_match('/\/api\/v\d+\/(\w+)/', $path, $matches);
        return $matches[1] ?? 'index';
    }
    
    /**
     * Format response based on Accept header
     */
    private function formatResponse(array $data, string $format): string
    {
        if ($format === 'xml') {
            return $this->toXML($data);
        }
        return json_encode($data);
    }
    
    /**
     * Generate error response
     */
    private function error(string $message, int $code, array $extra = []): array
    {
        return [
            'status' => 'error',
            'code' => $code,
            'message' => $message,
            'extra' => $extra,
            'version' => $this->defaultVersion
        ];
    }
}
```

### Step 3: Versioned Handler Implementation

```php
<?php
// /var/www/html/whmcs/includes/api/Versions/V2/Clients.php

namespace WHMCS\API\Versions\V2;

use WHMCS\User\Client;

class Clients
{
    private $deprecationWarnings = [];
    
    /**
     * Get clients list
     */
    public function index(array $params): array
    {
        $query = Client::query();
        
        // Apply filters
        if (!empty($params['status'])) {
            $query->where('status', $params['status']);
        }
        
        if (!empty($params['limit'])) {
            $query->limit((int) $params['limit']);
        }
        
        $clients = $query->get();
        
        return [
            'status' => 'success',
            'data' => [
                'clients' => $clients->map(function($client) {
                    return $this->formatClient($client);
                }),
                'count' => $clients->count()
            ],
            'meta' => [
                'version' => 'v2',
                'timestamp' => time()
            ]
        ];
    }
    
    /**
     * Get single client
     */
    public function show(array $params): array
    {
        $clientId = $params['client_id'] ?? null;
        
        if (!$clientId) {
            return ['status' => 'error', 'message' => 'client_id required'];
        }
        
        $client = Client::find($clientId);
        
        if (!$client) {
            return ['status' => 'error', 'message' => 'Client not found', 'code' => 404];
        }
        
        return [
            'status' => 'success',
            'data' => $this->formatClient($client, true),
            'meta' => [
                'version' => 'v2',
                'timestamp' => time()
            ]
        ];
    }
    
    /**
     * Create client
     */
    public function create(array $params): array
    {
        // Validation
        $required = ['firstname', 'lastname', 'email'];
        foreach ($required as $field) {
            if (empty($params[$field])) {
                return [
                    'status' => 'error',
                    'message' => "Missing required field: {$field}"
                ];
            }
        }
        
        // Check for duplicate email
        if (Client::where('email', $params['email'])->exists()) {
            return [
                'status' => 'error',
                'message' => 'Email already exists',
                'code' => 409
            ];
        }
        
        $client = Client::create([
            'firstname' => $params['firstname'],
            'lastname' => $params['lastname'],
            'email' => $params['email'],
            'companyname' => $params['companyname'] ?? '',
            'groupid' => $params['group_id'] ?? 0,
        ]);
        
        return [
            'status' => 'success',
            'data' => [
                'client_id' => $client->id,
                'created' => true
            ],
            'meta' => [
                'version' => 'v2'
            ]
        ];
    }
    
    /**
     * Format client for API response
     */
    private function formatClient(Client $client, bool $full = false): array
    {
        $data = [
            'id' => $client->id,
            'first_name' => $client->firstname,
            'last_name' => $client->lastname,
            'email' => $client->email,
            'status' => $client->status,
            'created_at' => $client->createdat
        ];
        
        if ($full) {
            $data['company'] = $client->companyname;
            $data['phone'] = $client->phonenumber;
            $data['address'] = [
                'address1' => $client->address1,
                'address2' => $client->address2,
                'city' => $client->city,
                'state' => $client->state,
                'country' => $client->country,
                'postcode' => $client->postcode
            ];
        }
        
        return $data;
    }
}
```

### Step 4: V1 to V2 Migration Handler

```php
<?php
// /var/www/html/whmcs/includes/api/Versions/V1/Clients.php
// Legacy v1 implementation for backward compatibility

namespace WHMCS\API\Versions\V1;

class Clients
{
    private $deprecationMessage = 'API v1 is deprecated. Please migrate to v2.';
    
    /**
     * Get clients list (v1 format)
     */
    public function index(array $params): array
    {
        // Log deprecation warning
        logActivity('API v1 called: ' . json_encode($params));
        
        // Convert v1 response to v2 format with warnings
        $v2Result = (new \WHMCS\API\Versions\V2\Clients())->index($params);
        
        // Add deprecation header
        header('X-API-Deprecation: ' . $this->deprecationMessage);
        header('X-API-Sunset-Date: 2025-12-31');
        
        // Wrap in v1 response format for compatibility
        return [
            'result' => $v2Result['status'],
            'clients' => $v2Result['data']['clients'],
            'deprecation_warning' => $this->deprecationMessage,
            'migration_guide' => 'https://docs.example.com/api/v1-to-v2-migration'
        ];
    }
    
    /**
     * Field mapping from v1 to v2
     */
    public function getFieldMapping(): array
    {
        return [
            'clientid' => 'id',
            'firstname' => 'first_name',
            'lastname' => 'last_name',
            'email' => 'email',
            'company' => 'company',
            'phone' => 'phone',
            'created' => 'created_at'
        ];
    }
}
```

### Step 5: API Version Deprecation Manager

```php
<?php
// /var/www/html/whmcs/includes/api/DeprecationManager.php

namespace WHMCS\API;

class DeprecationManager
{
    private $versions = [
        'v1' => [
            'released' => '2023-01-01',
            'deprecated' => '2024-06-01',
            'sunset' => '2025-12-31',
            'status' => 'deprecated',
            'migration_path' => 'v2'
        ],
        'v2' => [
            'released' => '2024-01-01',
            'deprecated' => null,
            'sunset' => null,
            'status' => 'current',
            'migration_path' => null
        ]
    ];
    
    /**
     * Check if version is deprecated
     */
    public function isDeprecated(string $version): bool
    {
        $versionInfo = $this->versions[$version] ?? null;
        
        if (!$versionInfo || $versionInfo['status'] !== 'deprecated') {
            return false;
        }
        
        return true;
    }
    
    /**
     * Check if version is sunset
     */
    public function isSunset(string $version): bool
    {
        $versionInfo = $this->versions[$version] ?? null;
        
        if (!$versionInfo || !$versionInfo['sunset']) {
            return false;
        }
        
        return strtotime($versionInfo['sunset']) < time();
    }
    
    /**
     * Add deprecation headers to response
     */
    public function addDeprecationHeaders(string $version): void
    {
        if (headers_sent()) {
            return;
        }
        
        $versionInfo = $this->versions[$version] ?? null;
        
        if ($versionInfo && $versionInfo['status'] === 'deprecated') {
            header('X-API-Deprecation: This API version is deprecated');
            header('X-API-Sunset-Date: ' . $versionInfo['sunset']);
            header('X-API-Migration-Guide: https://docs.example.com/api/v2');
            header('Retry-After: 86400');
        }
    }
    
    /**
     * Get version status
     */
    public function getVersionStatus(string $version): array
    {
        return $this->versions[$version] ?? [
            'status' => 'unknown',
            'message' => 'Version not found'
        ];
    }
    
    /**
     * Send migration notification
     */
    public function notifyDeprecation(string $email): bool
    {
        $body = "
            Dear API User,
            
            Your application is using a deprecated API version.
            
            Please review our migration guide:
            https://docs.example.com/api/v1-to-v2-migration
            
            Sunset Date: " . $this->versions['v1']['sunset'] . "
            
            Best regards,
            WHMCS API Team
        ";
        
        return mail($email, '[WHMCS] API Version Deprecation Notice', $body);
    }
}
```

### Step 6: API Documentation Generator

```yaml
# /var/www/html/whmcs/api-spec.yaml
# OpenAPI Specification for WHMCS API

openapi: 3.0.3
info:
  title: WHMCS API
  version: '2.0'
  description: WHMCS REST API for client management and billing

servers:
  - url: https://whmcs.example.com/api/v2
    description: Production
  - url: https://staging.whmcs.example.com/api/v2
    description: Staging

paths:
  /clients:
    get:
      summary: List clients
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [Active, Inactive, Suspended]
        - name: limit
          in: query
          schema:
            type: integer
            default: 50
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ClientList'

components:
  schemas:
    Client:
      type: object
      properties:
        id:
          type: integer
        first_name:
          type: string
        last_name:
          type: string
        email:
          type: string
        status:
          type: string
        created_at:
          type: string
          format: date-time

    ClientList:
      type: object
      properties:
        clients:
          type: array
          items:
            $ref: '#/components/schemas/Client'
        count:
          type: integer

  headers:
    X-API-Version:
      schema:
        type: string
      description: API version being used
```

## API Versioning Best Practices

1. **Semantic Versioning**: Use major.minor.patch format
2. **Backward Compatibility**: Maintain compatibility within major versions
3. **Clear Deprecation Path**: Provide 6-12 months notice before sunset
4. **Migration Documentation**: Clear guides for version migration
5. **Version in Headers**: Use Accept header for negotiation
6. **Feature Flags**: Use query parameters for gradual feature rollout
7. **Monitor Usage**: Track API version adoption

## Common Pitfalls

- **Breaking Changes**: Introducing breaking changes in minor versions
- **No Deprecation Notice**: Sunset without warning
- **Missing Migration Path**: No clear path from old to new versions
- **Inconsistent Naming**: Different naming conventions across versions
- **Over-versioning**: Creating too many versions for minor changes

## Verification Checklist

- [ ] Version routing implemented
- [ ] Versioned handlers created
- [ ] Backward compatibility maintained
- [ ] Deprecation notices added
- [ ] Migration documentation created
- [ ] Version monitoring in place
- [ ] Sunset dates configured

## Related Documentation

- [WHMCS API Development Workflow](whmcs-api-development-workflow.md)
- [WHMCS API Rate Limiting](whmcs-api-rate-limiting.md)
- [WHMCS API Authentication](whmcs-api-authentication.md)