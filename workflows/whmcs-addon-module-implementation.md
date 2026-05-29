# WHMCS Addon Module Implementation

## Overview

This workflow guides you through creating a WHMCS addon module. Addon modules extend WHMCS functionality with custom features, pages, and integrations.

## Prerequisites

- WHMCS v8.0+
- PHP 7.4+
- Basic understanding of MVC patterns
- Database access knowledge

## Step-by-Step Instructions

### Step 1: Create Module Structure

```
/modules/addons/YourAddon/
    ├── youraddon.php              # Main module file
    ├── Controller/
    │   ├── AdminController.php
    │   └── ClientController.php
    ├── Model/
    │   └── YourModel.php
    ├── views/
    │   ├── admin/
    │   │   └── dashboard.tpl
    │   └── client/
    │       └── overview.tpl
    ├── assets/
    │   ├── css/
    │   │   └── style.css
    │   └── js/
    │       └── script.js
    ├── hooks.php                  # Hook registration
    └── lang/
        └── english.php
```

### Step 2: Create Main Module File

Create `youraddon.php`:

```php
<?php
/**
 * WHMCS Addon Module - YourAddon
 *
 * @copyright Copyright (c) 2024 Your Name
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Module metadata.
 *
 * @return array
 */
function YourAddon_MetaData()
{
    return [
        'DisplayName' => 'Your Addon Name',
        'Description' => 'Description of what this addon does',
        'Author' => 'Your Name',
        'Version' => '1.0.0',
        'License' => 'Apache 2.0',
        'Fields' => [
            'license_key' => [
                'Type' => 'text',
                'Size' => '40',
                'Default' => '',
                'Description' => 'Your license key',
            ],
        ],
    ];
}

/**
 * Module activation.
 *
 * @return array
 */
function YourAddon_activate()
{
    try {
        // Create database tables
        $pdo = \WHMCS\Database\Capsule::connection()->getPdo();

        $pdo->exec("
            CREATE TABLE IF NOT EXISTS `mod_youraddon_data` (
                `id` INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `client_id` INT(11) DEFAULT NULL,
                `data` TEXT,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                INDEX `idx_client_id` (`client_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");

        $pdo->exec("
            CREATE TABLE IF NOT EXISTS `mod_youraddon_settings` (
                `setting_key` VARCHAR(100) NOT NULL PRIMARY KEY,
                `setting_value` TEXT,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");

        // Create cron for data processing
        $cronOutput = \WHMCS\Scheduling\Schedule::addCronJob([
            'name' => 'YourAddon Sync',
            'description' => 'Sync data with external service',
            'frequency' => 'hourly',
            'callback' => 'YourAddon_cronHandler',
            'runOnUpdate' => false,
        ]);

        // Set activation date
        setModuleConfig('youraddon', 'activated_at', date('Y-m-d H:i:s'));

        return [
            'status' => 'success',
            'description' => 'Addon activated successfully',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed to activate: ' . $e->getMessage(),
        ];
    }
}

/**
 * Module deactivation.
 *
 * @return array
 */
function YourAddon_deactivate()
{
    try {
        $pdo = \WHMCS\Database\Capsule::connection()->getPdo();

        // Optionally preserve data - comment out to delete
        // $pdo->exec("DROP TABLE IF EXISTS `mod_youraddon_data`");
        // $pdo->exec("DROP TABLE IF EXISTS `mod_youraddon_settings`");

        \WHMCS\Scheduling\Schedule::deleteCronJob('YourAddon Sync');

        return [
            'status' => 'success',
            'description' => 'Addon deactivated successfully',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed to deactivate: ' . $e->getMessage(),
        ];
    }
}

/**
 * Upgrade module.
 *
 * @param array $params
 * @return array
 */
function YourAddon_upgrade($params)
{
    $currentVersion = $params['version'];

    try {
        $pdo = \WHMCS\Database\Capsule::connection()->getPdo();

        if (version_compare($currentVersion, '1.1.0', '<')) {
            // Add new columns for v1.1
            $pdo->exec("
                ALTER TABLE `mod_youraddon_data`
                ADD COLUMN `extra_field` VARCHAR(255) DEFAULT NULL
                AFTER `data`
            ");
        }

        if (version_compare($currentVersion, '1.2.0', '<')) {
            // Add new table for v1.2
            $pdo->exec("
                CREATE TABLE IF NOT EXISTS `mod_youraddon_logs` (
                    `id` INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
                    `action` VARCHAR(50),
                    `details` TEXT,
                    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
                ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
            ");
        }

        return [
            'status' => 'success',
            'description' => "Upgraded to version {$params['newVersion']}",
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Upgrade failed: ' . $e->getMessage(),
        ];
    }
}

/**
 * Admin area output.
 *
 * @param array $params
 * @return string
 */
function YourAddon_output($params)
{
    // Determine which tab/action to display
    $action = $_REQUEST['action'] ?? 'dashboard';

    // Check permission
    if (!function_exists('checkPermission') || !checkPermission('YourAddon_Access')) {
        return '<div class="alert alert-warning">Access denied</div>';
    }

    // Route to appropriate controller
    switch ($action) {
        case 'settings':
            return YourAddon_adminSettings($params);

        case 'reports':
            return YourAddon_adminReports($params);

        case 'sync':
            return YourAddon_adminSync($params);

        case 'dashboard':
        default:
            return YourAddon_adminDashboard($params);
    }
}

/**
 * Admin dashboard view.
 *
 * @param array $params
 * @return string
 */
function YourAddon_adminDashboard($params)
{
    // Get statistics
    $stats = [
        'total_records' => \WHMCS\Database\Capsule::table('mod_youraddon_data')->count(),
        'active_today' => \WHMCS\Database\Capsule::table('mod_youraddon_data')
            ->whereDate('created_at', date('Y-m-d'))
            ->count(),
    ];

    // Recent activity
    $recentActivity = \WHMCS\Database\Capsule::table('mod_youraddon_data')
        ->orderBy('created_at', 'desc')
        ->limit(10)
        ->get();

    $smarty = new \WHMCS\Smarty\Frontend();
    $smarty->assign('stats', $stats);
    $smarty->assign('recentActivity', $recentActivity);
    $smarty->assign('moduleLink', $params['modulelink']);

    return $smarty->view('addon:youraddon/admin/dashboard');
}

/**
 * Admin settings view.
 *
 * @param array $params
 * @return string
 */
function YourAddon_adminSettings($params)
{
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        // Process form submission
        $settings = [
            'api_endpoint' => $_POST['api_endpoint'] ?? '',
            'sync_interval' => (int)($_POST['sync_interval'] ?? 60),
            'enable_logging' => ($_POST['enable_logging'] ?? 'off') === 'on',
        ];

        foreach ($settings as $key => $value) {
            setModuleConfig('youraddon', $key, $value);
        }

        return '<div class="alert alert-success">Settings saved successfully</div>';
    }

    $currentSettings = [
        'api_endpoint' => getModuleConfig('youraddon', 'api_endpoint') ?: 'https://api.example.com',
        'sync_interval' => getModuleConfig('youraddon', 'sync_interval') ?: 60,
        'enable_logging' => getModuleConfig('youraddon', 'enable_logging') ?: false,
    ];

    $smarty = new \WHMCS\Smarty\Frontend();
    $smarty->assign('settings', $currentSettings);
    $smarty->assign('moduleLink', $params['modulelink']);

    return $smarty->view('addon:youraddon/admin/settings');
}

/**
 * Admin reports view.
 *
 * @param array $params
 * @return string
 */
function YourAddon_adminReports($params)
{
    $dateFrom = $_GET['date_from'] ?? date('Y-m-01');
    $dateTo = $_GET['date_to'] ?? date('Y-m-d');

    $records = \WHMCS\Database\Capsule::table('mod_youraddon_data')
        ->whereBetween('created_at', [$dateFrom, $dateTo])
        ->orderBy('created_at', 'desc')
        ->limit(100)
        ->get();

    $smarty = new \WHMCS\Smarty\Frontend();
    $smarty->assign('records', $records);
    $smarty->assign('dateFrom', $dateFrom);
    $smarty->assign('dateTo', $dateTo);

    return $smarty->view('addon:youraddon/admin/reports');
}

/**
 * Manual sync action.
 *
 * @param array $params
 * @return string
 */
function YourAddon_adminSync($params)
{
    try {
        // Perform sync operation
        $api = new \YourAddon\ApiClient($params);
        $result = $api->syncData();

        return '<div class="alert alert-success">Sync completed. ' . $result['count'] . ' records updated.</div>';
    } catch (\Exception $e) {
        return '<div class="alert alert-danger">Sync failed: ' . $e->getMessage() . '</div>';
    }
}

/**
 * Client area output.
 *
 * @param array $params
 * @return array
 */
function YourAddon_clientarea($params)
{
    $client = Menu::context();

    return [
        'pagetitle' => 'Your Addon',
        'breadcrumb' => ['Your Addon'],
        'templatefile' => 'client/overview',
        'requirelogin' => true,
        'vars' => [
            'clientData' => YourAddon_getClientData($client->id),
        ],
    ];
}

/**
 * Get client-specific data.
 *
 * @param int $clientId
 * @return array
 */
function YourAddon_getClientData($clientId)
{
    $data = \WHMCS\Database\Capsule::table('mod_youraddon_data')
        ->where('client_id', $clientId)
        ->first();

    return $data ? json_decode($data->data, true) : [];
}

/**
 * Sidebar widget for admin dashboard.
 *
 * @param array $params
 * @return string
 */
function YourAddon_sidebar($params)
{
    $stats = [
        'pending' => \WHMCS\Database\Capsule::table('mod_youraddon_data')
            ->where('status', 'pending')
            ->count(),
    ];

    $smarty = new \WHMCS\Smarty\Frontend();
    $smarty->assign('stats', $stats);

    return $smarty->view('addon:youraddon/sidebar');
}

/**
 * Cron handler callback.
 *
 * @param array $params
 */
function YourAddon_cronHandler($params)
{
    try {
        $api = new \YourAddon\ApiClient($params);
        $result = $api->syncData();

        logActivity("YourAddon sync completed: {$result['count']} records");
    } catch (\Exception $e) {
        logActivity("YourAddon sync failed: " . $e->getMessage());
    }
}
```

### Step 3: Create Hooks File

Create `hooks.php`:

```php
<?php
/**
 * YourAddon Hooks Registration
 */

use WHMCS\View\Menu\Item as MenuItem;

// Add client area navigation
add_hook('ClientAreaSecondaryNavbar', 1, function(MenuItem $secondaryNavbar) {
    $client = Menu::context();

    if ($client) {
        $navItem = $secondaryNavbar->addChild('youraddon', [
            'label' => 'Your Addon',
            'uri' => 'index.php?m=youraddon',
            'icon' => 'fa-cube',
        ]);
    }
});

// Process after client login
add_hook('ClientLogin', 1, function($vars) {
    logActivity("YourAddon: Client {$vars['userid']} logged in");
});

// Process new order
add_hook('OrderPaid', 1, function($vars) {
    // Trigger any post-purchase actions
    $addon = new \YourAddon\ClientHandler();
    $addon->onOrderPlaced($vars['orderid']);
});

// Add invoice item calculation
add_hook('InvoiceCreationPreOutput', 1, function($vars) {
    // Modify invoice before display
});

// Add widget to admin dashboard
add_hook('AdminHomeWidgets', 1, function() {
    return [
        'name' => 'YourAddon Stats',
        'filename' => 'addon:youraddon/widgets/stats.php',
        'columns' => 1,
        'primary' => true,
    ];
});
```

### Step 4: Create Admin Template

Create `views/admin/dashboard.tpl`:

```smarty
<div class="panel panel-default">
    <div class="panel-heading">
        <i class="fa fa-cube"></i> Your Addon Dashboard
        <div class="pull-right">
            <a href="{$moduleLink}&action=settings" class="btn btn-xs btn-default">
                <i class="fa fa-cog"></i> Settings
            </a>
            <a href="{$moduleLink}&action=sync" class="btn btn-xs btn-primary">
                <i class="fa fa-refresh"></i> Sync Now
            </a>
        </div>
    </div>
    <div class="panel-body">
        <div class="row">
            <div class="col-sm-4">
                <div class="stat-box">
                    <div class="stat-value">{$stats.total_records}</div>
                    <div class="stat-label">Total Records</div>
                </div>
            </div>
            <div class="col-sm-4">
                <div class="stat-box">
                    <div class="stat-value">{$stats.active_today}</div>
                    <div class="stat-label">Added Today</div>
                </div>
            </div>
            <div class="col-sm-4">
                <div class="stat-box">
                    <div class="stat-value">--</div>
                    <div class="stat-label">Last Sync</div>
                </div>
            </div>
        </div>

        <h4>Recent Activity</h4>
        <table class="table table-striped">
            <thead>
                <tr>
                    <th>ID</th>
                    <th>Client</th>
                    <th>Date</th>
                    <th>Status</th>
                </tr>
            </thead>
            <tbody>
                {foreach $recentActivity as $activity}
                <tr>
                    <td>{$activity.id}</td>
                    <td>{$activity.client_id}</td>
                    <td>{$activity.created_at}</td>
                    <td><span class="label label-success">Active</span></td>
                </tr>
                {foreachelse}
                <tr>
                    <td colspan="4" class="text-center">No recent activity</td>
                </tr>
                {/foreach}
            </tbody>
        </table>
    </div>
</div>
```

## Expected Outcomes

- Addon appears in WHMCS addon list
- Activation creates required database tables
- Admin dashboard displays statistics
- Settings page saves configuration
- Client area shows addon interface

## Testing Checklist

- [ ] Module activates successfully
- [ ] Database tables created
- [ ] Activation returns success status
- [ ] Deactivation works correctly
- [ ] Upgrade processes migration
- [ ] Admin dashboard loads
- [ ] Settings save/load correctly
- [ ] Client area accessible
- [ ] Hooks execute on triggers
- [ ] Cron job runs on schedule
