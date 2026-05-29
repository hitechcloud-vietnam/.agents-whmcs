# WHMCS Addon Module API

Complete reference for addon module development in WHMCS.

## Module Structure

### Basic Addon Module Template

```php
<?php
/**
 * WHMCS Addon Module
 * 
 * Addon: Your Addon
 * Version: 1.0
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function youraddon_MetaData()
{
    return [
        'DisplayName' => 'Your Addon Name',
        'Description' => 'Description of what the addon does',
        'Version' => '1.0',
        'Author' => 'Your Name',
        'Parameters' => [],
    ];
}

function youraddon_ConfigArray()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Your Addon',
        ],
        'ApiKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '50',
        ],
        'EnableLogging' => [
            'FriendlyName' => 'Enable Logging',
            'Type' => 'yesno',
        ],
    ];
}
```

## Client Area Functions

### clientarea()

```php
/**
 * Client area output
 */
function youraddon_clientarea(array $params)
{
    $smarty = new WHMCS\Smarty();
    
    // Check if user is logged in
    if (!$_SESSION['uid']) {
        return [
            'pagetitle' => 'Login Required',
            'breadcrumb' => ['Login'],
            'templatefile' => 'login_required',
            'vars' => [
                'message' => 'Please log in to access this feature.',
            ],
        ];
    }
    
    // Determine which action to show
    $action = $_GET['action'] ?? 'overview';
    
    switch ($action) {
        case 'overview':
            return youraddon_overview($params);
        
        case 'settings':
            return youraddon_settings($params);
        
        case 'history':
            return youraddon_history($params);
        
        default:
            return youraddon_overview($params);
    }
}

function youraddon_overview(array $params): array
{
    $clientId = $_SESSION['uid'];
    
    // Get addon data
    $addonData = Capsule::table('mod_youraddon_data')
        ->where('userid', $clientId)
        ->first();
    
    return [
        'pagetitle' => 'Your Addon',
        'breadcrumb' => ['Your Addon'],
        'templatefile' => 'overview',
        'vars' => [
            'title' => 'Your Addon Overview',
            'data' => $addonData,
            'status' => $addonData->status ?? 'inactive',
        ],
    ];
}
```

### clientareaHeaderTabs()

```php
/**
 * Add tabs to client area navigation
 */
function youraddon_clientareaHeaderTabs()
{
    if (!$_SESSION['uid']) {
        return [];
    }
    
    return [
        'Your Addon' => [
            'label' => 'Your Addon',
            'uri' => 'addon.php?module=youraddon',
            'attributes' => [
                'class' => 'your-addon-tab',
            ],
        ],
    ];
}
```

## Admin Area Functions

### adminoutput()

```php
/**
 * Admin area output
 */
function youraddon_adminoutput(array $params)
{
    global $aInt;
    
    $aInt->requireAuth(1); // Require administrator authentication
    
    $smarty = new WHMCS\Smarty();
    
    $action = $_GET['action'] ?? 'dashboard';
    
    switch ($action) {
        case 'dashboard':
            return youraddon_adminDashboard($params, $smarty);
        
        case 'settings':
            return youraddon_adminSettings($params, $smarty);
        
        case 'reports':
            return youraddon_adminReports($params, $smarty);
        
        default:
            return youraddon_adminDashboard($params, $smarty);
    }
}

function youraddon_adminDashboard(array $params, $smarty): array
{
    // Get statistics
    $totalUsers = Capsule::table('mod_youraddon_data')->count();
    $activeUsers = Capsule::table('mod_youraddon_data')
        ->where('status', 'active')
        ->count();
    
    $recentActivity = Capsule::table('mod_youraddon_activity')
        ->orderBy('created_at', 'desc')
        ->limit(10)
        ->get();
    
    return [
        'pagetitle' => 'Your Addon - Dashboard',
        'breadcrumb' => ['Home', 'Your Addon'],
        'templatefile' => 'admin/dashboard',
        'vars' => [
            'total_users' => $totalUsers,
            'active_users' => $activeUsers,
            'recent_activity' => $recentActivity,
        ],
    ];
}
```

### adminMenu()

```php
/**
 * Add admin menu items
 */
function youraddon_adminMenu()
{
    return [
        'Your Addon' => [
            'label' => 'Your Addon',
            'uri' => 'addon.php?module=youraddon',
            'description' => 'Manage your addon',
        ],
    ];
}
```

## Service Integration

### serviceEdit()

```php
/**
 * Service configuration page
 */
function youraddon_serviceEdit(array $params)
{
    $serviceId = $params['serviceid'];
    
    // Get addon configuration
    $config = Capsule::table('mod_youraddon_service_config')
        ->where('service_id', $serviceId)
        ->first();
    
    return [
        'templatefile' => 'service_config',
        'vars' => [
            'config' => $config,
            'service_id' => $serviceId,
        ],
    ];
}
```

### serviceSave()

```php
/**
 * Save service configuration
 */
function youraddon_serviceSave(array $params)
{
    $serviceId = $params['serviceid'];
    $data = $params['config'];
    
    Capsule::table('mod_youraddon_service_config')
        ->updateOrInsert(
            ['service_id' => $serviceId],
            [
                'data' => json_encode($data),
                'updated_at' => date('Y-m-d H:i:s'),
            ]
        );
    
    return ['success' => true];
}
```

## Hook Integration

### activate()

```php
/**
 * Module activation callback
 */
function youraddon_activate()
{
    // Create database tables
    if (!Capsule::schema()->hasTable('mod_youraddon_data')) {
        Capsule::schema()->create('mod_youraddon_data', function($table) {
            $table->increments('id');
            $table->integer('userid')->unsigned();
            $table->string('status', 50)->default('inactive');
            $table->text('data')->nullable();
            $table->timestamps();
        });
    }
    
    if (!Capsule::schema()->hasTable('mod_youraddon_activity')) {
        Capsule::schema()->create('mod_youraddon_activity', function($table) {
            $table->increments('id');
            $table->integer('userid')->unsigned();
            $table->string('action', 100);
            $table->text('details')->nullable();
            $table->timestamp('created_at')->nullable();
        });
    }
    
    // Register hooks
    add_hook('ClientAreaPrimaryNavbar', 1, function($navbar) {
        $navbar->addChild('your-addon', [
            'label' => 'Your Addon',
            'uri' => 'addon.php?module=youraddon',
            'order' => 99,
        ]);
    });
    
    return [
        'status' => 'success',
        'description' => 'Your Addon has been activated successfully.',
    ];
}
```

### deactivate()

```php
/**
 * Module deactivation callback
 */
function youraddon_deactivate()
{
    // Clean up if needed
    // Note: Tables are not automatically dropped
    
    return [
        'status' => 'success',
        'description' => 'Your Addon has been deactivated.',
    ];
}
```

## Usage Examples

### Create addon configuration page

```php
<?php
// admin/templates/youraddon/settings.tpl
<div class="whmcs-header">
    <h2>{$pagetitle}</h2>
</div>

<div class="panel panel-default">
    <div class="panel-heading">
        Addon Settings
    </div>
    <div class="panel-body">
        <form method="post" action="addon.php?module=youraddon&action=save_settings">
            <div class="form-group">
                <label>API Key</label>
                <input type="password" name="api_key" class="form-control" 
                       value="{$settings.api_key}">
            </div>
            <div class="form-group">
                <label>Enable Debug Mode</label>
                <input type="checkbox" name="debug_mode" value="1"
                       {if $settings.debug_mode}checked{/if}>
            </div>
            <button type="submit" class="btn btn-primary">Save Settings</button>
        </form>
    </div>
</div>
```

## Related Documentation

- [whmcs-module-lifecycle.md](whmcs-module-lifecycle.md)
- [whmcs-module-configuration-api.md](whmcs-module-configuration-api.md)