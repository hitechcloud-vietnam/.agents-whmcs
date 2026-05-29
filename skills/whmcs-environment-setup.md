# WHMCS Environment Setup

## Skill Description
Set up development, staging, and production environments for WHMCS modules with proper configuration management.

## Prerequisites
- WHMCS installation
- Multiple server environments
- Configuration management

## Step-by-Step Implementation

### 1. Environment Configuration
```php
<?php
// config/environment.php

return [
    'env' => getenv('APP_ENV') ?: 'production',

    'debug' => (bool) getenv('APP_DEBUG'),

    'database' => [
        'host' => getenv('DB_HOST') ?: 'localhost',
        'database' => getenv('DB_NAME') ?: 'whmcs',
        'username' => getenv('DB_USER') ?: 'root',
        'password' => getenv('DB_PASSWORD') ?: '',
        'charset' => 'utf8mb4',
        'collation' => 'utf8mb4_unicode_ci'
    ],

    'cache' => [
        'driver' => getenv('CACHE_DRIVER') ?: 'file',
        'prefix' => 'whmcs_module_'
    ],

    'logging' => [
        'level' => getenv('LOG_LEVEL') ?: 'error',
        'path' => __DIR__ . '/../logs'
    ],

    'api' => [
        'rate_limit' => (int) getenv('API_RATE_LIMIT') ?: 60,
        'timeout' => (int) getenv('API_TIMEOUT') ?: 30
    ]
];
```

### 2. Environment-Specific Settings
```php
<?php
// config/local.php
return [
    'debug' => true,
    'logging' => ['level' => 'debug']
];
```

```php
<?php
// config/production.php
return [
    'debug' => false,
    'logging' => ['level' => 'error']
];
```

### 3. Environment Detection
```php
<?php
// bootstrap.php

function getEnvironment(): string
{
    $environments = [
        'local' => ['localhost', '127.0.0.1'],
        'staging' => ['staging.'],
        'production' => ['.']
    ];

    $host = $_SERVER['HTTP_HOST'] ?? '';

    foreach ($environments as $env => $patterns) {
        foreach ($patterns as $pattern) {
            if (strpos($host, $pattern) !== false) {
                return $env;
            }
        }
    }

    return getenv('APP_ENV') ?: 'production';
}

function loadEnvironmentConfig(): void
{
    $env = getEnvironment();

    $configFile = __DIR__ . '/config/' . $env . '.php';

    if (file_exists($configFile)) {
        $config = include $configFile;
        foreach ($config as $key => $value) {
            if (!defined($key)) {
                define($key, $value);
            }
        }
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Config not loading | Check environment detection |
| Missing variables | Use defaults |
| Permissions | Set correct file permissions |

## Security Considerations

1. **Don't commit secrets** - Use .gitignore
2. **Use environment variables** - Store sensitive data
3. **Validate configuration** - Check on startup

## Testing Checklist

- [ ] Test local environment
- [ ] Test staging environment
- [ ] Test production environment

## Reference Links

- [Environment Configuration](https://12factor.net/config)
