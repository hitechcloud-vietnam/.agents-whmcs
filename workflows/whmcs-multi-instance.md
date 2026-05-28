# WHMCS Multi-Instance Management Workflow

## Purpose

Guide to managing multiple WHMCS instances for different brands, departments, or use cases, including deployment, synchronization, and unified management.

## Prerequisites

- Multiple WHMCS licenses or enterprise deployment
- Shared database or dedicated instances
- Centralized management tools
- Understanding of multi-tenant architecture

## Workflow Steps

### Step 1: Instance Architecture Planning

Design multi-instance structure:

```php
// Architecture configurations

/**
 * Instance Types:
 * 1. Independent Instances - Separate databases, separate brands
 * 2. Shared Database - Multiple brands, shared clients
 * 3. Hub-Spoke - Central management with child instances
 */

// Configuration for shared management
return [
    'instances' => [
        'primary' => [
            'name' => 'Primary WHMCS',
            'path' => '/var/www/whmcs-primary',
            'url' => 'https://billing-primary.example.com',
            'database' => 'whmcs_primary',
            'role' => 'master', // Central management
        ],
        'brand_a' => [
            'name' => 'Brand A',
            'path' => '/var/www/whmcs-brand-a',
            'url' => 'https://billing-branda.example.com',
            'database' => 'whmcs_brand_a',
            'role' => 'child',
            'sync_enabled' => true,
        ],
        'brand_b' => [
            'name' => 'Brand B',
            'path' => '/var/www/whmcs-brand-b',
            'url' => 'https://billing-brandb.example.com',
            'database' => 'whmcs_brand_b',
            'role' => 'child',
            'sync_enabled' => true,
        ],
        'internal' => [
            'name' => 'Internal Tools',
            'path' => '/var/www/whmcs-internal',
            'url' => 'https://internal.example.com/whmcs',
            'database' => 'whmcs_internal',
            'role' => 'isolated', // No sync
        ],
    ],
    
    'shared_config' => [
        'products' => true, // Share product catalog
        'clients' => false, // Independent client databases
        'admin_users' => false, // Separate admin accounts
    ],
];
```

### Step 2: Instance Provisioning Script

Automate new instance creation:

```php
// scripts/provision_instance.php

class InstanceProvisioner
{
    private $basePath;
    private $dbHost;
    private $dbUser;
    private $dbPass;
    
    public function __construct(array $config)
    {
        $this->basePath = $config['base_path'];
        $this->dbHost = $config['db_host'];
        $this->dbUser = $config['db_user'];
        $this->dbPass = $config['db_pass'];
    }
    
    /**
     * Provision new WHMCS instance
     */
    public function provision(array $instanceConfig): array
    {
        $instanceName = $instanceConfig['name'];
        $instancePath = "{$this->basePath}/{$instanceName}";
        
        // Step 1: Create directory structure
        $this->createDirectories($instancePath);
        
        // Step 2: Clone base installation
        $this->cloneBaseInstallation($instanceConfig['source'], $instancePath);
        
        // Step 3: Create database
        $this->createDatabase($instanceConfig['database']);
        
        // Step 4: Generate configuration
        $this->generateConfig($instancePath, $instanceConfig);
        
        // Step 5: Configure instance-specific settings
        $this->configureInstance($instancePath, $instanceConfig);
        
        // Step 6: Install base data
        $this->installBaseData($instanceConfig);
        
        return [
            'success' => true,
            'path' => $instancePath,
            'url' => $instanceConfig['url'],
            'admin_url' => $instanceConfig['url'] . '/admin',
        ];
    }
    
    private function createDirectories(string $path): void
    {
        $dirs = [
            $path,
            "{$path}/storage",
            "{$path}/storage/logs",
            "{$path}/storage/cache",
            "{$path}/storage/uploads",
            "{$path}/templates_c",
        ];
        
        foreach ($dirs as $dir) {
            if (!is_dir($dir)) {
                mkdir($dir, 0755, true);
            }
        }
        
        // Set permissions
        exec("chmod -R 755 {$path}");
        exec("chmod 550 {$path}/includes");
    }
    
    private function cloneBaseInstallation(string $source, string $destination): void
    {
        // Rsync for efficient copying
        exec("rsync -a --exclude='storage/*' --exclude='templates_c/*' " .
             "--exclude='vendor/*' {$source}/ {$destination}/");
    }
    
    private function createDatabase(string $databaseName): void
    {
        $pdo = new PDO(
            "mysql:host={$this->dbHost}",
            $this->dbUser,
            $this->dbPass
        );
        
        $pdo->exec("CREATE DATABASE IF NOT EXISTS `{$databaseName}` 
                    CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci");
        
        // Create limited user for this database
        $user = $databaseName . '_user';
        $pass = bin2hex(random_bytes(16));
        
        $pdo->exec("CREATE USER IF NOT EXISTS '{$user}'@'localhost' 
                    IDENTIFIED BY '{$pass}'");
        $pdo->exec("GRANT ALL PRIVILEGES ON {$databaseName}.* 
                    TO '{$user}'@'localhost'");
        $pdo->exec("FLUSH PRIVILEGES");
    }
    
    private function generateConfig(string $path, array $config): void
    {
        $dbName = $config['database'];
        $dbUser = $dbName . '_user';
        $dbPass = bin2hex(random_bytes(16));
        
        $cc_encryption_hash = bin2hex(random_bytes(32));
        
        $configContent = <<<PHP
<?php
# WHMCS Configuration - {$config['name']}
# Generated: {$this->getTimestamp()}

\$whmcs_base_uri = '{$config['url']}/';
\$current_dir = getcwd();

\$db_host = '{$this->dbHost}';
\$db_username = '{$dbUser}';
\$db_password = '{$dbPass}';
\$db_name = '{$dbName}';

\$cc_encryption_hash = '{$cc_encryption_hash}';

\$templates_compiledir = \$current_dir . '/templates_c/';
\$bulk_domain_template = \$current_dir . '/resources/domainpricing/templates/bulk-template.csv';
\$api_access_key = bin2hex(random_bytes(24));

// Instance configuration
define('INSTANCE_ID', '{$config['name']}');
define('INSTANCE_TYPE', '{$config['role']}');
define('INSTANCE_URL', '{$config['url']}');

// Disable XML-RPC for security
define('DISABLE_XMLRPC', true);

// Performance settings
\$display_errors = false;
PHP;
        
        file_put_contents("{$path}/includes/config.php", $configContent);
        chmod("{$path}/includes/config.php", 0440);
    }
    
    private function configureInstance(string $path, array $config): void
    {
        // Set instance-specific values
        $values = [
            'CompanyName' => $config['name'],
            'SystemURL' => $config['url'],
            'SystemSSLURL' => $config['url'],
            'Domain' => parse_url($config['url'], PHP_URL_HOST),
            'LogoURL' => "/assets/{$config['name']}/logo.png",
            'Template' => $config['template'] ?? 'default',
        ];
        
        foreach ($values as $setting => $value) {
            Capsule::table('tblconfiguration')
                ->updateOrInsert(
                    ['setting' => $setting],
                    ['value' => $value]
                );
        }
    }
}
```

### Step 3: Centralized Management

Implement unified management across instances:

```php
// modules/addons/multi_instance_manager/manager.php

/**
 * Central management dashboard
 */
function mim_config(): array
{
    return [
        'name' => 'Multi-Instance Manager',
        'description' => 'Manage multiple WHMCS instances from one dashboard',
        'version' => '1.0',
    ];
}

class MultiInstanceManager
{
    private $instances = [];
    
    public function __construct()
    {
        $this->loadInstanceConfig();
    }
    
    /**
     * Execute command on remote instance
     */
    public function remoteCommand(string $instance, string $command, array $params = []): array
    {
        $config = $this->instances[$instance];
        
        if (!$config || $config['role'] === 'isolated') {
            return ['error' => 'Instance not accessible'];
        }
        
        $url = $config['url'] . '/api/internal/command.php';
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode([
                'command' => $command,
                'params' => $params,
                'auth_token' => $this->getAuthToken($instance),
            ]),
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true);
    }
    
    /**
     * Sync products to child instances
     */
    public function syncProducts(string $instance, array $productIds = []): array
    {
        $products = Capsule::table('tblproducts')
            ->whereIn('id', $productIds)
            ->get()
            ->toArray();
        
        $result = $this->remoteCommand($instance, 'import_products', [
            'products' => $products,
        ]);
        
        return [
            'synced' => count($products),
            'result' => $result,
        ];
    }
    
    /**
     * Get status of all instances
     */
    public function getAllStatuses(): array
    {
        $statuses = [];
        
        foreach ($this->instances as $name => $config) {
            $statuses[$name] = [
                'online' => $this->checkInstanceHealth($config),
                'last_sync' => $this->getLastSyncTime($name),
                'version' => $this->getInstanceVersion($config),
                'client_count' => $this->getClientCount($config),
            ];
        }
        
        return $statuses;
    }
    
    private function checkInstanceHealth(array $config): bool
    {
        $ch = curl_init($config['url'] . '/api/health.php');
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 5);
        curl_setopt($ch, CURLOPT_HTTPHEADER, [
            'Authorization: Bearer ' . $this->getAuthToken($config['name'])
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return $httpCode === 200;
    }
    
    private function getAuthToken(string $instance): string
    {
        return hash('sha256', INSTANCE_SECRET . $instance);
    }
}
```

### Step 4: Data Synchronization

Synchronize data across instances:

```php
/**
 * Client data sync between instances
 */
class InstanceDataSync
{
    private $sourceInstance;
    private $targetInstance;
    
    /**
     * Sync client to child instance
     */
    public function syncClient(int $clientId, string $targetInstance): array
    {
        // Get client data from source
        $client = Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();
        
        $contacts = Capsule::table('tblcontacts')
            ->where('userid', $clientId)
            ->get()
            ->toArray();
        
        $customFields = Capsule::table('tblcustomfieldsvalues')
            ->where('relid', $clientId)
            ->where('type', 'client')
            ->get()
            ->toArray();
        
        // Create on target instance
        $targetClientId = $this->createClientOnTarget($targetInstance, $client);
        
        // Sync contacts
        foreach ($contacts as $contact) {
            $this->createContactOnTarget($targetInstance, $contact, $targetClientId);
        }
        
        // Sync custom fields
        $this->syncCustomFields($targetInstance, $customFields, $targetClientId, 'client');
        
        return [
            'success' => true,
            'target_client_id' => $targetClientId,
        ];
    }
    
    /**
     * Sync service to child instance
     */
    public function syncService(int $serviceId, string $targetInstance, int $targetClientId): array
    {
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();
        
        $addons = Capsule::table('tblhostingaddons')
            ->where('hostingid', $serviceId)
            ->get()
            ->toArray();
        
        $customFields = Capsule::table('tblcustomfieldsvalues')
            ->where('relid', $serviceId)
            ->where('type', 'service')
            ->get()
            ->toArray();
        
        // Create on target
        $serviceData = (array)$service;
        $serviceData['userid'] = $targetClientId;
        unset($serviceData['id']);
        
        $targetServiceId = Capsule::table('tblhosting')->insertGetId($serviceData);
        
        // Sync addons
        foreach ($addons as $addon) {
            $addonData = (array)$addon;
            $addonData['hostingid'] = $targetServiceId;
            unset($addonData['id']);
            Capsule::table('tblhostingaddons')->insert($addonData);
        }
        
        return [
            'success' => true,
            'target_service_id' => $targetServiceId,
        ];
    }
}
```

### Step 5: Unified Reporting

Generate consolidated reports:

```php
/**
 * Generate unified report across all instances
 */
function generateUnifiedReport(): array
{
    $manager = new MultiInstanceManager();
    $report = [
        'generated_at' => date('Y-m-d H:i:s'),
        'instances' => [],
    ];
    
    foreach ($manager->getAllStatuses() as $instanceName => $status) {
        $report['instances'][$instanceName] = [
            'status' => $status['online'] ? 'online' : 'offline',
            'version' => $status['version'],
            'clients' => $status['client_count'],
            'services' => $manager->remoteCommand($instanceName, 'count_services'),
            'revenue' => $manager->remoteCommand($instanceName, 'get_total_revenue', [
                'period' => 'month',
            ]),
        ];
    }
    
    // Aggregate totals
    $report['totals'] = [
        'clients' => array_sum(array_column($report['instances'], 'clients')),
        'services' => array_sum(array_column($report['instances'], 'services')),
        'revenue' => array_sum(array_column($report['instances'], 'revenue')),
    ];
    
    return $report;
}
```

## Best Practices

1. **Use consistent versions** - Keep all instances on same WHMCS version
2. **Centralize management** - Use API for remote management
3. **Automate deployments** - Script new instance provisioning
4. **Monitor all instances** - Unified monitoring dashboard
5. **Document differences** - Track instance-specific customizations
6. **Plan for scale** - Consider resource requirements
7. **Secure inter-instance communication** - Use authentication tokens
8. **Regular health checks** - Automated uptime monitoring

## Common Pitfalls to Avoid

1. **Version mismatches** - Causes compatibility issues
2. **Manual management** - Errors and inconsistencies
3. **Missing documentation** - Hard to troubleshoot
4. **Insufficient resources** - Slow performance across instances
5. **Unsecured communication** - Data exposure between instances
6. **Duplicate data** - Confusing when synced incorrectly
7. **No testing environment** - Production issues
8. **Ignoring dependencies** - Some data cannot be shared
