# WHMCS SAML Integration

## Overview

Comprehensive guide for saml integration in WHMCS. This documentation covers implementation, configuration, and best practices.

## Technical Details

### Architecture

The saml integration system in WHMCS provides essential functionality for managing various aspects of your hosting business operations.

### Key Components

| Component | Description | Priority |
|-----------|-------------|----------|
| Core Engine | Main processing logic | High |
| API Layer | External communication | High |
| Storage | Data persistence | High |
| Cache Layer | Performance optimization | Medium |

### System Requirements

- WHMCS Version: 8.0+
- PHP Version: 7.4+ / 8.x
- Database: MySQL 5.7+ / MariaDB 10.3+
- Memory: Minimum 512MB RAM

## Code Examples

### Basic Implementation

```php
<?php
/**
 * SAML Integration implementation
 *
 * @package WHMCS\Module\SAMLIntegration
 */

namespace WHMCS\Module;

class SAMLIntegration
{
    protected $config;
    protected $logger;

    public function __construct(array $config = [])
    {
        $this->config = $config;
        $this->logger = \WHMCS\Log\Log::factory('saml_integration');
    }

    /**
     * Initialize the saml integration process
     */
    public function initialize(): bool
    {
        if (!$this->validateConfig()) {
            return false;
        }

        $this->logger->info('Initializing SAML Integration');
        return true;
    }

    /**
     * Validate configuration
     */
    protected function validateConfig(): bool
    {
        return !empty($this->config['api_key'] ?? '');
    }

    /**
     * Process the main operation
     */
    public function process(): array
    {
        try {
            $result = $this->executeOperation();
            return ['success' => true, 'data' => $result];
        } catch (\Exception $e) {
            $this->logger->error('Operation failed: ' . $e->getMessage());
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    /**
     * Execute the core operation
     */
    protected function executeOperation(): mixed
    {
        // Core processing logic
        return ['status' => 'completed'];
    }
}
```

### Configuration Handler

```php
<?php
/**
 * Configuration page for SAML Integration
 */
function saml_integration_config($vars)
{
    return [
        'name' => 'SAML Integration',
        'description' => 'Configure saml integration settings for WHMCS',
        'version' => '1.0.0',
        'fields' => [
            'enabled' => [
                'Type' => 'yesno',
                'FriendlyName' => 'Enable Feature',
                'Description' => 'Enable or disable saml integration',
                'Default' => 'yes',
            ],
            'api_key' => [
                'Type' => 'password',
                'FriendlyName' => 'API Key',
                'Description' => 'Enter your API key',
                'Size' => '50',
            ],
            'api_secret' => [
                'Type' => 'password',
                'FriendlyName' => 'API Secret',
                'Description' => 'Enter your API secret',
                'Size' => '50',
            ],
            'environment' => [
                'Type' => 'dropdown',
                'FriendlyName' => 'Environment',
                'Options' => 'Production,Sandbox',
                'Default' => 'Production',
                'Description' => 'Select the API environment',
            ],
            'timeout' => [
                'Type' => 'text',
                'FriendlyName' => 'Timeout (seconds)',
                'Description' => 'API request timeout',
                'Default' => '30',
                'Size' => '10',
            ],
            'debug_mode' => [
                'Type' => 'yesno',
                'FriendlyName' => 'Debug Mode',
                'Description' => 'Enable detailed logging',
                'Default' => 'no',
            ],
            'webhook_url' => [
                'Type' => 'text',
                'FriendlyName' => 'Webhook URL',
                'Description' => 'Callback URL for webhooks',
                'Size' => '80',
            ],
        ],
    ];
}
```

### Hook Integration

```php
<?php
/**
 * Register hooks for saml integration
 */
add_hook('samlintegration', 1, function($vars) {
    $config = get_config('saml_integration');

    if (!$config['enabled']) {
        return;
    }

    $handler = new \WHMCS\Module\SAMLIntegration($config);
    return $handler->process();
});

// Optional: Register cron hook
add_hook('DailyCronJob', 1, function($vars) {
    $handler = new \WHMCS\Module\SAMLIntegration();
    $handler->runScheduledTasks();
});
```

### API Client Example

```php
<?php
class SAMLIntegrationApiClient
{
    private string $baseUrl;
    private string $apiKey;
    private int $timeout;
    private \GuzzleHttp\Client $httpClient;

    public function __construct(array $config)
    {
        $this->baseUrl = $config['base_url'] ?? 'https://api.example.com';
        $this->apiKey = $config['api_key'] ?? '';
        $this->timeout = $config['timeout'] ?? 30;

        $this->httpClient = new \GuzzleHttp\Client([
            'base_uri' => $this->baseUrl,
            'timeout' => $this->timeout,
            'headers' => [
                'Authorization' => 'Bearer ' . $this->apiKey,
                'Content-Type' => 'application/json',
                'Accept' => 'application/json',
            ],
        ]);
    }

    public function request(string $method, string $endpoint, array $data = []): array
    {
        try {
            $response = $this->httpClient->request($method, $endpoint, [
                'json' => $data,
            ]);

            return json_decode($response->getBody()->getContents(), true);
        } catch (\GuzzleHttp\Exception\GuzzleException $e) {
            logModuleCall('saml_integration', $method, $endpoint, $data, $e->getMessage());
            throw new \Exception('API request failed: ' . $e->getMessage());
        }
    }
}
```

## Configuration Options

### Available Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| enabled | boolean | false | Enable/disable feature |
| api_key | string | - | API authentication key |
| api_secret | string | - | API secret for signing |
| environment | dropdown | Production | API environment |
| timeout | integer | 30 | Request timeout in seconds |
| debug_mode | boolean | false | Enable debug logging |
| webhook_url | string | - | Webhook callback URL |

### Advanced Configuration

```php
<?php
function saml_integration_activate()
{
    $schema = Capsule::schema();

    // Create settings table
    $schema->create('mod_saml_integration_settings', function($table) {
        $table->increments('id');
        $table->string('setting_key', 100)->unique();
        $table->text('setting_value')->nullable();
        $table->boolean('is_encrypted')->default(false);
        $table->timestamps();
    });

    // Create log table
    $schema->create('mod_saml_integration_logs', function($table) {
        $table->increments('id');
        $table->string('level', 20);
        $table->text('message');
        $table->json('context')->nullable();
        $table->timestamp('created_at')->useCurrent();
    });

    return [
        'status' => 'success',
        'description' => 'SAML Integration has been activated successfully',
    ];
}

function saml_integration_deactivate()
{
    // Optional: Clean up data
    // Capsule::schema()->dropIfExists('mod_saml_integration_settings');
    // Capsule::schema()->dropIfExists('mod_saml_integration_logs');

    return [
        'status' => 'success',
        'description' => 'SAML Integration has been deactivated',
    ];
}

function saml_integration_upgrade($vars)
{
    $currentVersion = $vars['version'];

    if (version_compare($currentVersion, '1.1.0', '<')) {
        // Migration from 1.0 to 1.1
        Capsule::schema()->table('mod_saml_integration_settings', function($table) {
            $table->boolean('is_encrypted')->default(false)->after('setting_value');
        });
    }

    if (version_compare($currentVersion, '1.2.0', '<')) {
        // Migration from 1.1 to 1.2
        Capsule::table('mod_saml_integration_settings')->insert([
            ['setting_key' => 'new_setting', 'setting_value' => 'default_value'],
        ]);
    }
}
```

## Troubleshooting Tips

### Common Issues

#### 1. Configuration Not Saving

**Symptoms:**
- Settings revert after saving
- Database errors in logs

**Solutions:**
```php
// Verify table exists
if (!Capsule::schema()->hasTable('mod_saml_integration_settings')) {
    // Re-run activation
}

// Check permissions
$table = Capsule::schema()->getTable('mod_saml_integration_settings');
if (!$table) {
    throw new \Exception('Table not found');
}
```

#### 2. API Errors

**Symptoms:**
- Connection timeouts
- Authentication failures
- Invalid responses

**Solutions:**
```php
// Enable debug logging
$config = get_config('saml_integration');
if ($config['debug_mode']) {
    logModuleCall('saml_integration', __FUNCTION__, $params, $response, $error);
}

// Verify credentials
$client = new SAMLIntegrationApiClient($config);
try {
    $test = $client->request('GET', '/health');
} catch (\Exception $e) {
    // Handle error
}

// Check network connectivity
if (!fsockopen('api.example.com', 443, $errno, $errstr, 5)) {
    throw new \Exception('Network unreachable');
}
```

#### 3. Performance Issues

**Symptoms:**
- Slow response times
- High server load
- Memory exhausted

**Solutions:**
```php
// Implement caching
$cache = \WHMCS\Caching\CacheFactory::factory('custom');
$key = 'saml_integration_cache_' . md5(serialize($params));

if ($cached = $cache->get($key)) {
    return $cached;
}

$result = expensiveOperation();
$cache->set($key, $result, 300); // 5 minute cache

// Use pagination
$limit = 100;
$offset = ($page - 1) * $limit;
$results = Capsule::table('mod_saml_integration_data')
    ->limit($limit)
    ->offset($offset)
    ->get();

// Implement queue for heavy operations
dispatch(new \App\Jobs\SAMLIntegrationJob($data));
```

### Debug Mode

```php
<?php
function saml_integration_debug(string $message, array $context = [], string $level = 'info'): void
{
    $config = get_config('saml_integration');

    if (!($config['debug_mode'] ?? false)) {
        return;
    }

    $logEntry = [
        'timestamp' => date('Y-m-d H:i:s'),
        'level' => $level,
        'message' => $message,
        'context' => $context,
    ];

    \WHMCS\Database\Capsule::table('mod_saml_integration_logs')->insert($logEntry);

    // Also log to file
    $logFile = __DIR__ . '/logs/debug.log';
    file_put_contents(
        $logFile,
        json_encode($logEntry) . PHP_EOL,
        FILE_APPEND
    );
}
```

### Testing Checklist

- [ ] Module activates without errors
- [ ] Configuration saves and loads correctly
- [ ] All hooks fire as expected
- [ ] API calls succeed in test environment
- [ ] Error handling works correctly
- [ ] Logging captures necessary information
- [ ] Uninstall removes all module data
- [ ] Upgrade path works correctly
- [ ] Performance is acceptable under load

## Best Practices

### Security

- Always validate and sanitize input
- Use prepared statements for database queries
- Implement rate limiting for API calls
- Store sensitive data encrypted
- Use HTTPS for all external communications

### Performance

- Cache expensive operations
- Use pagination for large result sets
- Implement async processing for heavy tasks
- Optimize database queries with indexes
- Use connection pooling

### Maintainability

- Follow WHMCS coding standards
- Use namespaces for all classes
- Document complex logic
- Write unit tests
- Keep code modular

### Error Handling

- Return consistent error formats
- Log all operations with context
- Provide meaningful error messages
- Handle edge cases gracefully

## Related Documentation

- [WHMCS Module Development Guide](./whmcs-module-best-practices.md)
- [WHMCS Hook System](./whmcs-hook-system.md)
- [WHMCS API Documentation](./whmcs-api-classes.md)
- [WHMCS Security Standards](./whmcs-module-security.md)

## Changelog

### Version 1.0.0
- Initial release
- Basic functionality implemented
- Configuration system added
- Hook integration support

## Support

For issues and questions:
- Email: support@example.com
- Documentation: https://docs.example.com
- GitHub Issues: https://github.com/example/whmcs-module/issues
