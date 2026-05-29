# WHMCS Addon Module From Scratch Workflow

## Description
Create a custom WHMCS addon module from scratch with proper structure and best practices.

## Prerequisites
- WHMCS installation (latest version)
- PHP 8.1+ knowledge
- Basic understanding of WHMCS hooks
- Code editor (VS Code, PHPStorm)

## Module Structure
```
modules/
└── addons/
    └── clicodes_example/
        ├── clicodes_example.php        # Main module file
        ├── LICENSE                     # License file
        ├── README.md                    # Documentation
        ├── hooks.php                   # Hook functions
        ├── includes/
        │   └── Helper.php             # Helper classes
        └── templates/
            └── admin/
                └── example.tpl         # Admin templates
```

## Steps

### Step 1: Create Module Directory
```bash
mkdir -p /var/www/whmcs/modules/addons/clicodes_example
cd /var/www/whmcs/modules/addons/clicodes_example
```

### Step 2: Create Main Module File
```php
<?php
/**
 * WHMCS Addon Module - CLICodes Example
 *
 * @copyright Copyright (c) 2024 Your Company
 * @license https://example.com/license
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Define module configuration
 */
function clicodes_example_config()
{
    return [
        'name' => 'CLICodes Example Module',
        'description' => 'An example addon module demonstrating WHMCS module structure.',
        'author' => 'Your Company',
        'version' => '1.0.0',
        'language' => 'english',
        'fields' => [
            'apiKey' => [
                'Type' => 'text',
                'Default' => '',
                'Description' => 'Enter your API Key',
            ],
            'enableLogging' => [
                'Type' => 'yesno',
                'Default' => '1',
                'Description' => 'Enable debug logging',
            ],
            'webhookUrl' => [
                'Type' => 'text',
                'Default' => '',
                'Description' => 'Webhook URL for notifications',
            ],
        ],
    ];
}

/**
 * Activate the module
 */
function clicodes_example_activate()
{
    // Create database tables
    try {
        $pdo = Capsule::connection()->getPdo();
        
        $pdo->exec("
            CREATE TABLE IF NOT EXISTS `mod_clicodes_example` (
                `id` INT(11) NOT NULL AUTO_INCREMENT,
                `user_id` INT(11) DEFAULT NULL,
                `data` TEXT,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        
        return [
            'status' => 'success',
            'description' => 'Module activated successfully.',
        ];
    } catch (Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed to activate module: ' . $e->getMessage(),
        ];
    }
}

/**
 * Deactivate the module
 */
function clicodes_example_deactivate()
{
    try {
        $pdo = Capsule::connection()->getPdo();
        $pdo->exec("DROP TABLE IF EXISTS `mod_clicodes_example`");
        
        return [
            'status' => 'success',
            'description' => 'Module deactivated successfully.',
        ];
    } catch (Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed to deactivate module.',
        ];
    }
}

/**
 * Upgrade the module
 */
function clicodes_example_upgrade($vars)
{
    $version = $vars['version'];
    
    if ($version < '1.1.0') {
        // Add new column for v1.1.0
        try {
            Capsule::schema()->table('mod_clicodes_example', function($table) {
                $table->text('additional_data')->nullable()->change();
            });
        } catch (Exception $e) {
            logActivity("Upgrade failed: " . $e->getMessage());
        }
    }
}

/**
 * Output the admin area output
 */
function clicodes_example_output($vars)
{
    $action = isset($_REQUEST['action']) ? $_REQUEST['action'] : 'dashboard';
    
    // Check permissions
    if (!checkPermission('Configure Addon Modules', true)) {
        return '<div class="alert alert-danger">Access denied.</div>';
    }
    
    $smarty = new Smarty;
    $smarty->assign('modulename', 'clicodes_example');
    $smarty->assign('variables', $vars);
    $smarty->assign('modulelink', $vars['modulelink']);
    
    switch ($action) {
        case 'settings':
            $smarty->assign('settings', clicodes_example_getSettings($vars));
            $output = $smarty->fetch(dirname(__FILE__) . '/templates/admin/settings.tpl');
            break;
            
        case 'reports':
            $smarty->assign('reports', clicodes_example_getReports());
            $output = $smarty->fetch(dirname(__FILE__) . '/templates/admin/reports.tpl');
            break;
            
        default:
            $smarty->assign('stats', clicodes_example_getStats());
            $output = $smarty->fetch(dirname(__FILE__) . '/templates/admin/dashboard.tpl');
    }
    
    return $output;
}

/**
 * Sidebar output
 */
function clicodes_example_sidebar($vars)
{
    $sidebar = '<div class="panel panel-default">
        <div class="panel-heading">
            ' . $vars['lang']['moduleName'] . '
        </div>
        <div class="panel-body">
            <p>Quick links:</p>
            <ul>
                <li><a href="' . $vars['modulelink'] . '">Dashboard</a></li>
                <li><a href="' . $vars['modulelink'] . '&action=settings">Settings</a></li>
                <li><a href="' . $vars['modulelink'] . '&action=reports">Reports</a></li>
            </ul>
        </div>
    </div>';
    
    return $sidebar;
}

/**
 * Get module settings
 */
function clicodes_example_getSettings($vars)
{
    return [
        'apiKey' => $vars['apiKey'],
        'enableLogging' => $vars['enableLogging'] ?? false,
        'webhookUrl' => $vars['webhookUrl'] ?? '',
    ];
}

/**
 * Get statistics
 */
function clicodes_example_getStats()
{
    try {
        $result = Capsule::table('mod_clicodes_example')
            ->selectRaw('COUNT(*) as total')
            ->selectRaw('COUNT(CASE WHEN created_at > DATE_SUB(NOW(), INTERVAL 24 HOUR) THEN 1 END) as today')
            ->first();
        
        return [
            'total' => $result->total ?? 0,
            'today' => $result->today ?? 0,
        ];
    } catch (Exception $e) {
        return ['total' => 0, 'today' => 0];
    }
}

/**
 * Get reports
 */
function clicodes_example_getReports()
{
    return Capsule::table('mod_clicodes_example')
        ->orderBy('created_at', 'desc')
        ->limit(100)
        ->get();
}
```

### Step 3: Create Hooks File
```php
<?php
/**
 * WHMCS Hooks for CLICodes Example Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Hook: After client registration
 */
add_hook('ClientAreaRegistrationCompleted', 1, function($vars) {
    logActivity('New client registered: ' . $vars['userid']);
    
    // Store to module table
    Capsule::table('mod_clicodes_example')->insert([
        'user_id' => $vars['userid'],
        'data' => json_encode(['event' => 'registration', 'timestamp' => time()]),
    ]);
    
    // Log if enabled
    $config = Capsule::table('tbladdonmodules')
        ->where('module', 'clicodes_example')
        ->first();
    
    if ($config && $config->setting == 'enableLogging') {
        logModuleCall('clicodes_example', 'registration', $vars, 'Success');
    }
});

/**
 * Hook: After order placed
 */
add_hook('OrderPaid', 1, function($vars) {
    $orderId = $vars['orderId'];
    
    // Process the order data
    Capsule::table('mod_clicodes_example')->insert([
        'user_id' => $vars['userId'],
        'data' => json_encode(['event' => 'order_paid', 'order_id' => $orderId]),
    ]);
});

/**
 * Hook: Before email sending
 */
add_hook('EmailPreSend', 1, function($vars) {
    // Modify email content if needed
    if ($vars['type'] == 'invoice') {
        // Custom invoice processing
    }
});
```

### Step 4: Create Template Files
```smarty
{*
 * Admin Dashboard Template
 *}
<div class="module-header">
    <h3>{$variables.lang.dashboard}</h3>
</div>

<div class="row">
    <div class="col-sm-6">
        <div class="panel panel-default">
            <div class="panel-heading">
                Statistics
            </div>
            <div class="panel-body">
                <div class="stat">
                    <span class="stat-value">{$stats.total}</span>
                    <span class="stat-label">Total Records</span>
                </div>
                <div class="stat">
                    <span class="stat-value">{$stats.today}</span>
                    <span class="stat-label">Today</span>
                </div>
            </div>
        </div>
    </div>
    <div class="col-sm-6">
        <div class="panel panel-default">
            <div class="panel-heading">
                Quick Actions
            </div>
            <div class="panel-body">
                <a href="{$modulelink}&action=settings" class="btn btn-primary">
                    Configure Module
                </a>
                <a href="{$modulelink}&action=reports" class="btn btn-default">
                    View Reports
                </a>
            </div>
        </div>
    </div>
</div>

<style>
.stat { text-align: center; padding: 20px; }
.stat-value { display: block; font-size: 36px; font-weight: bold; }
.stat-label { color: #666; }
</style>
```

### Step 5: Create Activation/Deactivation Functions
```php
// Included in main file, shown here for clarity

function clicodes_example_activate()
{
    // Create required database tables
    $pdo = Capsule::connection()->getPdo();
    
    $pdo->exec("
        CREATE TABLE IF NOT EXISTS `mod_clicodes_example` (
            `id` INT(11) NOT NULL AUTO_INCREMENT,
            `user_id` INT(11) DEFAULT NULL,
            `data` TEXT,
            `settings` TEXT,
            `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
            `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
            PRIMARY KEY (`id`),
            KEY `idx_user_id` (`user_id`)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
    ");
    
    // Set default configuration
    Capsule::table('tbladdonmodules')->insert([
        'module' => 'clicodes_example',
        'setting' => 'enableLogging',
        'value' => '1',
    ]);
    
    return [
        'status' => 'success',
        'description' => 'Module activated successfully.',
    ];
}

function clicodes_example_deactivate()
{
    $pdo = Capsule::connection()->getPdo();
    
    // Drop tables
    $pdo->exec("DROP TABLE IF EXISTS `mod_clicodes_example`");
    
    // Remove configuration
    Capsule::table('tbladdonmodules')
        ->where('module', 'clicodes_example')
        ->delete();
    
    return [
        'status' => 'success',
        'description' => 'Module deactivated and cleaned up.',
    ];
}
```

### Step 6: Add Language Translations
```php
// Add to clicodes_example.php

function clicodes_example_lang($vars = [])
{
    return [
        'en' => [
            'moduleName' => 'CLICodes Example Module',
            'dashboard' => 'Dashboard',
            'settings' => 'Settings',
            'reports' => 'Reports',
            'saveSettings' => 'Save Settings',
            'settingsSaved' => 'Settings saved successfully.',
        ],
    ];
}
```

### Step 7: Test the Module
```bash
# 1. Copy module to WHMCS
cp -r clicodes_example /var/www/whmcs/modules/addons/

# 2. Set permissions
chown -R www-data:www-data /var/www/whmcs/modules/addons/clicodes_example
chmod -R 755 /var/www/whmcs/modules/addons/clicodes_example

# 3. Enable in WHMCS admin
# Go to: Configuration > Addon Modules
# Find CLICodes Example Module
# Click Activate
```

### Step 8: Debug Module
```php
// Add to config.php for debugging
error_reporting(E_ALL);
ini_set('display_errors', 1);

// Use logModuleCall for debugging
logModuleCall(
    'clicodes_example',
    'function_name',
    $input,
    $output,
    $result
);
```

## Best Practices
- Always use proper namespacing
- Use Capsule for database queries
- Implement proper error handling
- Use hooks for extensibility
- Add language support
- Write comprehensive README
- Follow WHMCS coding standards

## Tags
- addon-module
- development
- module-creation
- php
- whmcs