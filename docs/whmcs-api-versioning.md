# WHMCS API Versioning

## Overview

Understanding API versioning helps maintain compatibility when WHMCS releases updates.

## Version Strategy

### Current API Versions

| Version | WHMCS Version | Status |
|---------|---------------|--------|
| v1 | 7.x+ | Current |
| v2 | 8.x+ | Current |

### Version Detection

```php
<?php
class WhmcsApiVersion {
    public const VERSION_1 = 'v1';
    public const VERSION_2 = 'v2';
    
    public static function getCurrentVersion(): string
    {
        // Detect based on WHMCS installation
        $whmcsVersion = static::detectWhmcsVersion();
        
        if (version_compare($whmcsVersion, '8.0', '>=')) {
            return self::VERSION_2;
        }
        
        return self::VERSION_1;
    }
    
    private static function detectWhmcsVersion(): string
    {
        // Check from WHMCS constants
        if (defined('WHMCS_VERSION')) {
            return WHMCS_VERSION;
        }
        
        // Or from database
        $version = Capsule::table('tblconfiguration')
            ->where('setting', 'Version')
            ->value('value');
        
        return $version ?? '7.0';
    }
    
    public static function supportsFeature(string $feature): bool
    {
        $version = self::getCurrentVersion();
        
        $features = [
            'v2' => ['oauth2', 'webhooks', 'enhanced_pagination'],
            'v1' => ['basic_api', 'legacy_hooks'],
        ];
        
        return in_array($feature, $features[$version] ?? []);
    }
}
```

### Version-Aware Client

```php
<?php
class VersionAwareApiClient {
    private string $baseUrl;
    private string $version;
    private string $identifier;
    private string $secret;
    
    public function __construct(
        string $baseUrl,
        string $identifier,
        string $secret,
        ?string $version = null
    ) {
        $this->baseUrl = rtrim($baseUrl, '/');
        $this->identifier = $identifier;
        $this->secret = $secret;
        $this->version = $version ?? WhmcsApiVersion::getCurrentVersion();
    }
    
    public function request(string $action, array $params = []): array
    {
        $endpoint = $this->getEndpoint($action);
        $params = $this->prepareParams($action, $params);
        
        return $this->execute($endpoint, $params);
    }
    
    private function getEndpoint(string $action): string
    {
        $endpoints = [
            'v1' => $this->baseUrl . '/includes/api.php',
            'v2' => $this->baseUrl . '/api/v2/' . strtolower($action),
        ];
        
        return $endpoints[$this->version] ?? $endpoints['v1'];
    }
    
    private function prepareParams(string $action, array $params): array
    {
        $params['action'] = $action;
        $params['identifier'] = $this->identifier;
        $params['secret'] = $this->secret;
        $params['responsetype'] = 'json';
        
        // Transform params for version differences
        if ($this->version === self::VERSION_2) {
            $params = $this->transformForV2($action, $params);
        }
        
        return $params;
    }
    
    private function transformForV2(string $action, array $params): array
    {
        // V2 parameter transformations
        $transformations = [
            'clientid' => 'client_id',
            'invoiceid' => 'invoice_id',
            'serviceid' => 'service_id',
            'domainid' => 'domain_id',
        ];
        
        return array_combine(
            array_map(fn($k) => $transformations[$k] ?? $k, array_keys($params)),
            array_values($params)
        );
    }
    
    private function execute(string $endpoint, array $params): array
    {
        $ch = curl_init($endpoint);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($params),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        if ($httpCode >= 400) {
            throw new ApiException("HTTP {$httpCode}: {$response}");
        }
        
        return json_decode($response, true);
    }
    
    public function getVersion(): string
    {
        return $this->version;
    }
}
```

## Deprecation Management

```php
<?php
class DeprecationManager {
    private array $deprecatedEndpoints = [];
    private array $deprecatedParams = [];
    
    public function __construct()
    {
        $this->deprecatedEndpoints = [
            'GetClientPassword' => [
                'deprecated_at' => '2023-01-01',
                'removed_at' => '2024-01-01',
                'replacement' => 'GetClientSecurityQanswer',
                'version' => 'v2',
            ],
            'AddAnnouncement' => [
                'deprecated_at' => '2023-06-01',
                'removed_at' => '2025-01-01',
                'replacement' => 'CreateAnnouncement',
                'version' => 'v2',
            ],
        ];
        
        $this->deprecatedParams = [
            'GetClients' => [
                'limitstart' => [
                    'deprecated_at' => '2023-01-01',
                    'replacement' => 'page',
                    'version' => 'v2',
                ],
            ],
        ];
    }
    
    public function checkEndpoint(string $action): ?DeprecationInfo
    {
        if (isset($this->deprecatedEndpoints[$action])) {
            return new DeprecationInfo(
                $action,
                $this->deprecatedEndpoints[$action]
            );
        }
        return null;
    }
    
    public function checkParameter(string $action, string $param): ?DeprecationInfo
    {
        if (isset($this->deprecatedParams[$action][$param])) {
            return new DeprecationInfo(
                "{$action}.{$param}",
                $this->deprecatedParams[$action][$param]
            );
        }
        return null;
    }
    
    public function logUsage(string $action, array $params): void
    {
        $deprecation = $this->checkEndpoint($action);
        
        if ($deprecation) {
            Log::warning('Deprecated endpoint used', [
                'endpoint' => $action,
                'deprecated_at' => $deprecation->deprecatedAt,
                'will_be_removed' => $deprecation->removedAt,
                'replacement' => $deprecation->replacement,
                'params' => $params,
            ]);
        }
    }
}

class DeprecationInfo {
    public function __construct(
        public readonly string $target,
        public readonly array $info
    ) {}
    
    public function deprecatedAt(): string
    {
        return $this->info['deprecated_at'] ?? '';
    }
    
    public function removedAt(): string
    {
        return $this->info['removed_at'] ?? '';
    }
    
    public function replacement(): string
    {
        return $this->info['replacement'] ?? '';
    }
    
    public function isUrgent(): bool
    {
        $removedAt = strtotime($this->removedAt());
        $now = time();
        $daysRemaining = ($removedAt - $now) / (60 * 60 * 24);
        
        return $daysRemaining <= 30 && $daysRemaining > 0;
    }
}
```

## Migration Guide

### Upgrading from v1 to v2

```php
<?php
class V1ToV2Migrator {
    public function migrateRequest(string $action, array $params): array
    {
        $migrations = $this->getParamMigrations();
        
        if (isset($migrations[$action])) {
            return $this->transformParams($action, $params, $migrations[$action]);
        }
        
        return $params;
    }
    
    private function getParamMigrations(): array
    {
        return [
            'GetClient' => [
                'client_id' => 'clientid', // v2 param => v1 param
            ],
            'GetInvoice' => [
                'invoice_id' => 'invoiceid',
            ],
            'GetService' => [
                'service_id' => 'serviceid',
            ],
            'GetOrder' => [
                'order_id' => 'orderid',
            ],
        ];
    }
    
    private function transformParams(
        string $action,
        array $params,
        array $migration
    ): array {
        $transformed = [];
        
        foreach ($params as $key => $value) {
            if (isset($migration[$key])) {
                $transformed[$migration[$key]] = $value;
            } else {
                $transformed[$key] = $value;
            }
        }
        
        return $transformed;
    }
    
    public function migrateResponse(string $action, array $response): array
    {
        // Transform response fields if needed
        $transformations = $this->getResponseMigrations();
        
        if (isset($transformations[$action])) {
            return $this->transformResponse($response, $transformations[$action]);
        }
        
        return $response;
    }
    
    private function getResponseMigrations(): array
    {
        return [
            'GetClients' => [
                'client_id' => 'id', // v2 field => v1 field
            ],
        ];
    }
    
    private function transformResponse(array $response, array $transformation): array
    {
        foreach ($transformation as $newField => $oldField) {
            if (isset($response[$newField])) {
                $response[$oldField] = $response[$newField];
                unset($response[$newField]);
            }
        }
        
        return $response;
    }
}
```

## Feature Flags

```php
<?php
class FeatureFlags {
    private static ?array $flags = null;
    
    public static function isEnabled(string $feature, bool $default = false): bool
    {
        $flags = self::getFlags();
        return $flags[$feature] ?? $default;
    }
    
    private static function getFlags(): array
    {
        if (self::$flags !== null) {
            return self::$flags;
        }
        
        // Load from database or config
        $config = Capsule::table('tblconfiguration')
            ->where('setting', 'like', 'FeatureFlags_%')
            ->pluck('value', 'setting');
        
        self::$flags = [];
        foreach ($config as $setting => $value) {
            $flag = str_replace('FeatureFlags_', '', $setting);
            self::$flags[strtolower($flag)] = (bool) $value;
        }
        
        return self::$flags;
    }
    
    public static function isEnabledForVersion(
        string $feature,
        string $minVersion = '7.0'
    ): bool {
        $currentVersion = WhmcsApiVersion::getCurrentVersion();
        $currentNum = version_compare($currentVersion, 'v2', '>=') ? 2 : 1;
        $minNum = str_contains($minVersion, 'v2') ? 2 : 1;
        
        return $currentNum >= $minNum && self::isEnabled($feature);
    }
}

// Usage
if (FeatureFlags::isEnabled('oauth2_authentication')) {
    // Use OAuth 2.0
} else {
    // Fall back to API key auth
}

if (FeatureFlags::isEnabledForVersion('enhanced_webhooks', 'v2')) {
    // Use enhanced webhook features
}
```

## Best Practices

1. **Always specify version** - Pin to specific API versions in production
2. **Monitor deprecations** - Track deprecated endpoint usage
3. **Test migrations** - Verify behavior after version changes
4. **Use feature flags** - Enable new features progressively
5. **Keep changelog updated** - Document breaking changes

## Related Documentation

- [WHMCS API Authentication](/docs/whmcs-api-authentication.md)
- [WHMCS API Error Handling](/docs/whmcs-api-error-handling.md)