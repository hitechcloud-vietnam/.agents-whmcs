# WHMCS Addon Module Master

## Overview
Master skill for WHMCS addon module development. Covers module structure, admin/client area pages, database operations, and integration patterns.

## Addon Module Structure

```php
<?php
// /modules/addons/YourAddon/YourAddon.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Addon Module Information
 */
function YourAddon_config()
{
    return [
        'name' => 'Your Addon Name',
        'description' => 'Description of what this addon does',
        'author' => 'Your Name',
        'website' => 'https://yourwebsite.com',
        'email' => 'support@yourwebsite.com',
        'version' => '1.0.0',
        'fields' => [
            'apiKey' => [
                'FriendlyName' => 'API Key',
                'Type' => 'password',
                'Size' => '50',
                'Default' => '',
                'Description' => 'Your external API key',
            ],
            'setting1' => [
                'FriendlyName' => 'Setting One',
                'Type' => 'text',
                'Size' => '50',
                'Default' => 'default_value',
                'Description' => 'Description of this setting',
            ],
            'setting2' => [
                'FriendlyName' => 'Setting Two',
                'Type' => 'yesno',
                'Default' => 'on',
                'Description' => 'Enable feature X',
            ],
            'dropdown' => [
                'FriendlyName' => 'Dropdown Option',
                'Type' => 'dropdown',
                'Options' => [
                    'option1' => 'Option One',
                    'option2' => 'Option Two',
                    'option3' => 'Option Three',
                ],
                'Default' => 'option1',
            ],
        ],
    ];
}

/**
 * Activate Hook
 */
function YourAddon_activate()
{
    // Create database tables
    $sql = "CREATE TABLE IF NOT EXISTS `mod_youraddon_data` (
        `id` INT AUTO_INCREMENT PRIMARY KEY,
        `client_id` INT NULL,
        `data` TEXT,
        `status` VARCHAR(50) DEFAULT 'active',
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;";

    \Illuminate\Database\Capsule\Manager::statement($sql);

    // Create configuration
    set_config([
        'version' => '1.0.0',
        'activated_at' => date('Y-m-d H:i:s'),
    ]);

    // Return activation result
    return [
        'status' => 'success',
        'description' => 'Module activated successfully',
    ];
}

/**
 * Deactivate Hook
 */
function YourAddon_deactivate()
{
    // Clean up data (optional - preserve for reactivation)
    // \Illuminate\Database\Capsule\Manager::table('mod_youraddon_data')->truncate();

    // Remove configuration
    remove_config([
        'version',
        'activated_at',
    ]);

    return [
        'status' => 'success',
        'description' => 'Module deactivated successfully',
    ];
}

/**
 * Upgrade Hook
 */
function YourAddon_upgrade($vars)
{
    $oldVersion = $vars['version'];
    $newVersion = '1.0.0';

    if (version_compare($oldVersion, '1.0.0', '<')) {
        // Run migration from 0.x to 1.0.0
        \Illuminate\Database\Capsule\Manager::statement("
            ALTER TABLE `mod_youraddon_data` 
            ADD COLUMN `new_field` VARCHAR(255) DEFAULT NULL
        ");
    }

    // Update version in config
    set_config(['version' => $newVersion]);
}

/**
 * Output Buffering for Smarty
 */
function YourAddon_output($vars)
{
    $action = \App::getFromRequest('action');
    $modulename = 'youraddon';
    $filename = \App::getTempDir() . '/tpl_' . $modulename . '_' . session_id() . '.tpl';

    if ($vars['modulename'] === $modulename) {
        ob_start();
        yourAddonRenderTemplate($vars, $action);
        $contents = ob_get_clean();

        file_put_contents($filename, $contents);

        return $filename;
    }
}

/**
 * Sidebar Output
 */
function YourAddon_sidebar($vars)
{
    $modulename = 'youraddon';

    if ($vars['modulename'] === $modulename) {
        return [
            'DisplayName' => 'Your Addon',
            'Title' => '<span class="icon fa fa-plus-square"></span> Your Addon',
            'Uri' => 'addonmodules.php?module=youraddon',
            'Order' => 1,
        ];
    }
}

/**
 * Template Renderer
 */
function yourAddonRenderTemplate(array $vars, ?string $action)
{
    global $aInt;

    switch ($action) {
        case 'view':
            yourAddonRenderView($vars);
            break;

        case 'save':
            yourAddonHandleSave($vars);
            break;

        case 'process':
            yourAddonProcessAction($vars);
            break;

        case 'export':
            yourAddonExportData($vars);
            break;

        default:
            yourAddonRenderDashboard($vars);
    }
}

function yourAddonRenderDashboard(array $vars)
{
    global $aInt;

    // Get statistics
    $stats = [
        'total_records' => getRecordCount(),
        'active_records' => getActiveCount(),
        'total_value' => getTotalValue(),
    ];

    // Recent activity
    $recentActivity = getRecentActivity();

    echo '<div class="row">
        <div class="col-md-12">
            <h2>Your Addon Dashboard</h2>
        </div>
    </div>';

    echo '<div class="row">';
    foreach ($stats as $key => $value) {
        echo '<div class="col-md-4">
            <div class="panel panel-default">
                <div class="panel-heading">' . ucfirst(str_replace('_', ' ', $key)) . '</div>
                <div class="panel-body">
                    <h2>' . $value . '</h2>
                </div>
            </div>
        </div>';
    }
    echo '</div>';

    echo '<div class="row">
        <div class="col-md-12">
            <div class="panel panel-default">
                <div class="panel-heading">Recent Activity</div>
                <table class="table table-striped">
                    <thead>
                        <tr>
                            <th>Date</th>
                            <th>Client</th>
                            <th>Action</th>
                            <th>Status</th>
                        </tr>
                    </thead>
                    <tbody>';

    foreach ($recentActivity as $activity) {
        echo '<tr>
            <td>' . $activity['date'] . '</td>
            <td>' . $activity['client'] . '</td>
            <td>' . $activity['action'] . '</td>
            <td><span class="label label-' . $activity['status_class'] . '">' . $activity['status'] . '</span></td>
        </tr>';
    }

    echo '</tbody>
                </table>
            </div>
        </div>
    </div>';
}

/**
 * Client Area Output
 */
function YourAddon_clientarea($vars)
{
    $action = \App::getFromRequest('a');

    return [
        'pagetitle' => 'Your Addon',
        'breadcrumb' => ['index.php?m=youraddon' => 'Your Addon'],
        'templatefile' => 'clientarea',
        'vars' => [
            'someVar' => 'someValue',
        ],
    ];
}
```

## Admin Page Template

```php
<?php
// /modules/addons/YourAddon/templates/admin.tpl

<div class="row">
    <div class="col-md-12">
        <div class="panel panel-default">
            <div class="panel-heading">
                <i class="fa fa-cog"></i> Your Addon Configuration
            </div>
            <div class="panel-body">
                <form method="post" action="addonmodules.php?module=youraddon&action=save">
                    <input type="hidden" name="csrf_token" value="{$token}">

                    <div class="form-group">
                        <label for="setting1">Setting One</label>
                        <input type="text" class="form-control" id="setting1" name="settings[setting1]" 
                               value="{$settings.setting1}">
                    </div>

                    <div class="form-group">
                        <label>
                            <input type="checkbox" name="settings[setting2]" value="1" 
                                   {if $settings.setting2}checked{/if}>
                            Enable Feature
                        </label>
                    </div>

                    <div class="form-group">
                        <label for="dropdown">Dropdown Option</label>
                        <select class="form-control" id="dropdown" name="settings[dropdown]">
                            <option value="option1" {if $settings.dropdown == 'option1'}selected{/if}>Option One</option>
                            <option value="option2" {if $settings.dropdown == 'option2'}selected{/if}>Option Two</option>
                            <option value="option3" {if $settings.dropdown == 'option3'}selected{/if}>Option Three</option>
                        </select>
                    </div>

                    <button type="submit" class="btn btn-primary">
                        <i class="fa fa-save"></i> Save Changes
                    </button>
                </form>
            </div>
        </div>
    </div>
</div>

<div class="row">
    <div class="col-md-12">
        <div class="panel panel-default">
            <div class="panel-heading">
                <i class="fa fa-list"></i> Data Management
            </div>
            <div class="panel-body">
                <table class="table table-bordered">
                    <thead>
                        <tr>
                            <th>ID</th>
                            <th>Client</th>
                            <th>Data</th>
                            <th>Status</th>
                            <th>Created</th>
                            <th>Actions</th>
                        </tr>
                    </thead>
                    <tbody>
                        {foreach $records as $record}
                        <tr>
                            <td>{$record.id}</td>
                            <td>{$record.client_name}</td>
                            <td>{$record.data}</td>
                            <td>
                                <span class="label label-{$record.status_class}">{$record.status}</span>
                            </td>
                            <td>{$record.created_at}</td>
                            <td>
                                <a href="addonmodules.php?module=youraddon&action=view&id={$record.id}" 
                                   class="btn btn-xs btn-info">
                                    <i class="fa fa-eye"></i> View
                                </a>
                                <a href="addonmodules.php?module=youraddon&action=delete&id={$record.id}" 
                                   class="btn btn-xs btn-danger" onclick="return confirm('Are you sure?')">
                                    <i class="fa fa-trash"></i> Delete
                                </a>
                            </td>
                        </tr>
                        {/foreach}
                    </tbody>
                </table>

                {if $pagination.total_pages > 1}
                <div class="text-center">
                    <ul class="pagination">
                        {if $pagination.current_page > 1}
                        <li><a href="?module=youraddon&page={$pagination.current_page - 1}">&laquo;</a></li>
                        {/if}
                        {section name=i start=1 loop=$pagination.total_pages + 1}
                        <li class="{if $pagination.current_page == $smarty.section.i.index}active{/if}">
                            <a href="?module=youraddon&page={$smarty.section.i.index}">{$smarty.section.i.index}</a>
                        </li>
                        {/section}
                        {if $pagination.current_page < $pagination.total_pages}
                        <li><a href="?module=youraddon&page={$pagination.current_page + 1}">&raquo;</a></li>
                        {/if}
                    </ul>
                </div>
                {/if}
            </div>
        </div>
    </div>
</div>
```

## Client Area Template

```php
<?php
// /modules/addons/YourAddon/templates/clientarea.tpl

<div class="youraddon-client-area">
    <div class="panel panel-default">
        <div class="panel-heading">
            <h3 class="panel-title"><i class="fa fa-user"></i> Your Addon</h3>
        </div>
        <div class="panel-body">
            {if $clientLoggedIn}
                <div class="row">
                    <div class="col-md-12">
                        <p>Welcome back, {$client.firstname}!</p>

                        <div class="row">
                            <div class="col-md-4">
                                <div class="stat-box">
                                    <h3>{$stats.total}</h3>
                                    <p>Total Items</p>
                                </div>
                            </div>
                            <div class="col-md-4">
                                <div class="stat-box">
                                    <h3>{$stats.active}</h3>
                                    <p>Active Items</p>
                                </div>
                            </div>
                            <div class="col-md-4">
                                <div class="stat-box">
                                    <h3>{$stats.pending}</h3>
                                    <p>Pending Items</p>
                                </div>
                            </div>
                        </div>

                        <div class="mt-4">
                            <h4>Your Items</h4>
                            {if $items}
                            <table class="table">
                                <thead>
                                    <tr>
                                        <th>Item Name</th>
                                        <th>Status</th>
                                        <th>Created</th>
                                        <th>Actions</th>
                                    </tr>
                                </thead>
                                <tbody>
                                    {foreach $items as $item}
                                    <tr>
                                        <td>{$item.name}</td>
                                        <td>
                                            <span class="badge badge-{$item.status_class}">{$item.status}</span>
                                        </td>
                                        <td>{$item.created_at}</td>
                                        <td>
                                            <a href="?m=youraddon&action=view&id={$item.id}" 
                                               class="btn btn-sm btn-primary">
                                                View
                                            </a>
                                        </td>
                                    </tr>
                                    {/foreach}
                                </tbody>
                            </table>
                            {else}
                            <p class="text-muted">No items found.</p>
                            {/if}
                        </div>
                    </div>
                </div>
            {else}
                <div class="alert alert-warning">
                    Please log in to access this feature.
                </div>
            {/if}
        </div>
    </div>
</div>
```

## Helper Functions

```php
<?php
// /modules/addons/YourAddon/lib/Helper.php

namespace WHMCS\Addon\YourAddon;

class Helper
{
    public static function getSetting(string $key, $default = null)
    {
        $result = \Illuminate\Database\Capsule\Manager::table('tbladdonconfig')
            ->where('setting', $key)
            ->where('addon', 'youraddon')
            ->first();

        return $result ? $result->value : $default;
    }

    public static function setSetting(string $key, $value): void
    {
        \Illuminate\Database\Capsule\Manager::table('tbladdonconfig')
            ->updateOrInsert(
                ['addon' => 'youraddon', 'setting' => $key],
                ['value' => $value]
            );
    }

    public static function getClientRecords(int $clientId, array $options = [])
    {
        $query = \Illuminate\Database\Capsule\Manager::table('mod_youraddon_data')
            ->where('client_id', $clientId);

        if (isset($options['status'])) {
            $query->where('status', $options['status']);
        }

        return $query->get();
    }

    public static function createRecord(int $clientId, array $data): int
    {
        return \Illuminate\Database\Capsule\Manager::table('mod_youraddon_data')
            ->insertGetId([
                'client_id' => $clientId,
                'data' => json_encode($data),
                'status' => 'active',
                'created_at' => date('Y-m-d H:i:s'),
            ]);
    }

    public static function updateRecord(int $recordId, array $data): bool
    {
        return \Illuminate\Database\Capsule\Manager::table('mod_youraddon_data')
            ->where('id', $recordId)
            ->update($data);
    }

    public static function deleteRecord(int $recordId): bool
    {
        return \Illuminate\Database\Capsule\Manager::table('mod_youraddon_data')
            ->where('id', $recordId)
            ->delete() > 0;
    }

    public static function logActivity(string $message, array $context = []): void
    {
        logActivity("YourAddon: " . $message, $context);
    }

    public static function sendNotification(int $clientId, string $type, array $data = []): void
    {
        $client = \WHMCS\User\Client::find($clientId);

        if ($client) {
            send_email($type, $clientId, array_merge($data, [
                'client_name' => $client->fullName,
            ]));
        }
    }
}
```

## Best Practices

1. **Activation/Deactivation**: Always implement activation and deactivation hooks for proper cleanup
2. **Database Tables**: Create tables with proper prefixes and indexes
3. **Configuration**: Use WHMCS's built-in configuration storage
4. **Version Upgrades**: Implement upgrade hooks for smooth migrations
5. **Security**: Validate all inputs, use CSRF tokens, escape output
6. **Error Handling**: Wrap operations in try-catch and log errors
7. **Template Separation**: Keep business logic separate from templates
8. **Client/Admin Areas**: Implement both areas for full functionality
9. **Hooks Integration**: Integrate with WHMCS hooks for automation
10. **Documentation**: Document all functions and configuration options
