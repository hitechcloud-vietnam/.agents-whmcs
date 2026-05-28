# WHMCS Addon Module Developer Guide

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

Addon modules extend WHMCS functionality by adding custom features, integration with third-party services, and custom administrative interfaces. This guide covers the complete development process.

---

## Module Structure

```
modules/addons/
  your_addon/
    your_addon.php        # Main module file
    templates/
      admin.tpl           # Admin area template
      client.tpl          # Client area template
    assets/
      css/
        style.css
      js/
        script.js
      images/
        logo.png
    hooks/
      hook_functions.php  # Hook registrations
    lib/
      ApiClient.php       # API client library
      Helpers.php         # Helper functionsI
```

## Main Module File

```php
<?php
/**
 * Addon Module Definition
 * 
 * @package WHMCS
 * @subpackage AddonModule
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Module information
 * 
 * @return array Module metadata
 */
function your_addon_MetaData()
{
    return [
        'DisplayName'    => 'Your Addon Name',
        'Description'   => 'Description of what this addon does',
        'Author'         => 'Your Name',
        'Version'        => '1.0.0',
        'License'       => 'proprietary',
        'Languages'     => ['english'],
        'Requires'       => [
            'Php'        => '7.4',
            'Whmcs'      => '8.0',
        ],
    ];
}

/**
 * Get configuration
 * 
 * @return array Configuration fields
 */
function your_addon_config()
{
    return [
        'name' => [
            'Type'        => 'System',
            'Value'      => 'Your Addon Name',
        ],
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type'         => 'password',
            'Description' => 'Your API key from the service provider',
        ],
        'api_endpoint' => [
            'FriendlyName' => 'API Endpoint',
            'Type'         => 'text',
            'Size'         => '60',
            'Default'     => 'https://api.example.com',
        ],
        'enable_logging' => [
            'FriendlyName' => 'Enable Debug Logging',
            'Type'         => 'yesno',
        ],
        'webhook_secret' => [
            'FriendlyName' => 'Webhook Secret',
            'Type'         => 'password',
            'Description' => 'Secret for verifying webhook signatures',
        ],
    ];
}

/**
 * Activate module
 * 
 * @return string Success message
 */
function your_addon_activate()
{
    // Create database tables
    $createTableSql = "CREATE TABLE IF NOT EXISTS `mod_your_addon` (
        `id` INT(10) NOT NULL AUTO_INCREMENT PRIMARY KEY,
        `client_id` INT(10) NOT NULL,
        `external_id` VARCHAR(255) NULL,
        `settings` TEXT NULL,
        `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
        `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        INDEX `idx_client_id` (`client_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
    
    // Execute table creation
    $fulltext = "CREATE TABLE IF NOT EXISTS `mod_your_addon_data` (
        `id` INT(10) NOT NULL AUTO_INCREMENT PRIMARY KEY,
        `addon_id` INT(10) NOT NULL,
        `data_key` VARCHAR(100) NOT NULL,
        `data_value` TEXT NULL,
        FOREIGN KEY (`addon_id`) REFERENCES `mod_your_addon`(`id`) ON DELETE CASCADE,
        UNIQUE KEY `unique_key` (`addon_id`, `data_key`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
    
    full_query($createTableSql);
    full_query($fulltext);
    
    // Set activation config
    set_config_var('your_addon_version', '1.0.0');
    set_config_var('your_addon_activated', date('Y-m-d H:i:s'));
    
    return [
        'status'      => 'success',
        'description' => 'Addon activated successfully. Configure settings in the module configuration.',
    ];
}

/**
 * Deactivate module
 * 
 * @return string Success message
 */
function your_addon_deactivate()
{
    // Drop database tables
    $dropTableSql = "DROP TABLE IF EXISTS `mod_your_addon_data`";
    full_query($dropTableSql);
    
    $dropMainSql = "DROP TABLE IF EXISTS `mod_your_addon`";
    full_query($dropMainSql);
    
    // Remove config variables
    delete_config_var('your_addon_version');
    delete_config_var('your_addon_activated');
    
    return [
        'status'      => 'success',
        'description' => 'Addon deactivated and cleanup complete.',
    ];
}

/**
 * Upgrade module
 * 
 * @param string $fromVersion Current version
 * @param string $toVersion Target version
 */
function your_addon_upgrade($fromVersion, $toVersion)
{
    $versions = ['1.0.0', '1.1.0', '2.0.0'];
    
    foreach ($versions as $version) {
        if (version_compare($fromVersion, $version, '<') && 
            version_compare($version, $toVersion, '<=')) {
            
            switch ($version) {
                case '1.1.0':
                    // Add migration for 1.1.0
                    $sql = "ALTER TABLE `mod_your_addon` ADD COLUMN `new_field` VARCHAR(50) NULL";
                    full_query($sql);
                    break;
                    
                case '2.0.0':
                    // Add migration for 2.0.0
                    $sql = "CREATE TABLE IF NOT EXISTS `mod_your_addon_v2` (
                        `id` INT(10) NOT NULL AUTO_INCREMENT PRIMARY KEY,
                        `name` VARCHAR(100) NOT NULL
                    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
                    full_query($sql);
                    break;
            }
        }
    }
    
    update_config_var('your_addon_version', $toVersion);
}
```

## Admin Area Output

```php
/**
 * Output admin area sidebar panel
 * 
 * @return string HTML content
 */
function your_addon_sidebar()
{
    $stats = [
        'total'      => your_addon_countRecords(),
        'active'     => your_addon_countActiveRecords(),
        'pending'    => your_addon_countPendingRecords(),
    ];
    
    $sidebar = '<div class="module-sidebar-tick">';
    $sidebar .= '<div class="title">Your Addon</div>';
    $sidebar .= '<div class="item">Total: ' . $stats['total'] . '</div>';
    $sidebar .= '<div class="item">Active: ' . $stats['active'] . '</div>';
    $sidebar .= '<div class="item">Pending: ' . $stats['pending'] . '</div>';
    $sidebar .= '</div>';
    
    return $sidebar;
}

/**
 * Output admin area output
 * 
 * @return string HTML content
 */
function your_addon_output($vars)
{
    $action = $vars['action'] ?? 'dashboard';
    $moduleLink = $vars['modulelink'];
    
    // Check for permission
    if (!function_exists('checkPermission')) {
        require_once ROOTDIR . '/includes/adminfunctions.php';
    }
    
    if (!checkPermission('Your Addon Access', true)) {
        return '<div class="error-box">Access denied.</div>';
    }
    
    // Render based on action
    switch ($action) {
        case 'settings':
            return your_addon_renderSettings($vars);
            
        case 'view':
            return your_addon_renderDetails($vars);
            
        case 'sync':
            return your_addon_syncData($vars);
            
        default:
            return your_addon_renderDashboard($vars);
    }
}

/**
 * Render dashboard
 * 
 * @param array $vars Module variables
 * @return string HTML content
 */
function your_addon_renderDashboard($vars)
{
    $moduleLink = $vars['modulelink'];
    $records = your_addon_getRecentRecords(10);
    
    $html = '<div class="contentbox">';
    $html .= '<h2>Your Addon Dashboard</h2>';
    $html .= '<p><a href="' . $moduleLink . '&action=settings" class="btn">Configure</a></p>';
    $html .= '<table class="datatable">';
    $html .= '<thead><tr><th>ID</th><th>Client</th><th>Status</th><th>Actions</th></tr></thead>';
    $html .= '<tbody>';
    
    foreach ($records as $record) {
        $html .= '<tr>';
        $html .= '<td>' . $record['id'] . '</td>';
        $html .= '<td>' . htmlspecialchars($record['client_name']) . '</td>';
        $html .= '<td>' . ucfirst($record['status']) . '</td>';
        $html .= '<td><a href="' . $moduleLink . '&action=view&id=' . $record['id'] . '">View</a></td>';
        $html .= '</tr>';
    }
    
    $html .= '</tbody></table>';
    $html .= '</div>';
    
    return $html;
}
```

## Client Area Output

```php
/**
 * Client area output
 * 
 * @param array $vars Smarty variables
 * @return string HTML content
 */
function your_addon_clientarea($vars)
{
    // Check if user is logged in
    if (!$_SESSION['uid']) {
        return '<p>Please log in to access this feature.</p>';
    }
    
    $clientId = $_SESSION['uid'];
    $action = $vars['request']['action'] ?? 'home';
    $smarty = $vars['smarty'];
    
    // Load client data
    $clientData = your_addon_getClientData($clientId);
    
    switch ($action) {
        case 'connect':
            return your_addon_renderConnectPage($vars, $clientId);
            
        case 'settings':
            return your_addon_renderClientSettings($vars, $clientId);
            
        default:
            return your_addon_renderClientHome($vars, $clientData);
    }
}

/**
 * Render client home page
 * 
 * @param array $vars Smarty variables
 * @param array $clientData Client data
 * @return string HTML content
 */
function your_addon_renderClientHome($vars, $clientData)
{
    $modal = [];
    $modulelink = $vars['modulelink'];
    
    $modals['breadcrumb'] = [
        'title' => 'Your Addon',
        'links' => [
            'Home' => 'clientarea.php?action=account',
        ],
    ];
    
    return [
        'pagetitle'     => 'Your Addon',
        'breadcrumb'   => $modals['breadcrumb'],
        'template'     => 'your_addon/client-home.tpl',
        'requireLogin' => true,
        'vars'          => [
            'records'    => $clientData['records'] ?? [],
            'status'     => $clientData['status'] ?? 'inactive',
            'modulelink' => $modulelink,
        ],
    ];
}
```

## Database Operations

```php
/**
 * Get client data
 * 
 * @param int $clientId Client ID
 * @return array Client data
 */
function your_addon_getClientData($clientId)
{
    $result = select_query('mod_your_addon', '*', ['client_id' => $clientId]);
    $data = mysql_fetch_array($result, MYSQLI_ASSOC);
    
    if ($data) {
        // Decode settings JSON
        $data['settings'] = json_decode($data['settings'] ?? '{}', true);
        
        // Get related data
        $data['records'] = your_addon_getRecordsForClient($clientId);
    }
    
    return $data;
}

/**
 * Save client settings
 * 
 * @param int $clientId Client ID
 * @param array $settings Settings to save
 * @return bool Success status
 */
function your_addon_saveClientSettings($clientId, $settings)
{
    $settingsJson = json_encode($settings);
    
    $existing = select_query('mod_your_addon', 'id', ['client_id' => $clientId]);
    
    if (mysql_num_rows($existing) > 0) {
        update_query('mod_your_addon', 
            ['settings' => $settingsJson, 'updated_at' => 'NOW()'],
            ['client_id' => $clientId]
        );
    } else {
        insert_query('mod_your_addon', [
            'client_id' => $clientId,
            'settings'  => $settingsJson,
            'created_at' => 'NOW()',
        ]);
    }
    
    return true;
}

/**
 * Sync data with external service
 * 
 * @param int $clientId Client ID
 * @return array Sync result
 */
function your_addon_syncForClient($clientId)
{
    $clientData = your_addon_getClientData($clientId);
    
    if (empty($clientData['external_id'])) {
        // Create external record
        $externalData = [
            'client_id'  => $clientId,
            'email'      => get_query_val('tblclients', 'email', ['id' => $clientId]),
            'name'       => get_query_val('tblclients', 'firstname', ['id' => $clientId]) . ' ' . 
                           get_query_val('tblclients', 'lastname', ['id' => $clientId]),
        ];
        
        $response = your_addon_apiRequest('POST', '/clients', $externalData);
        
        if ($response['success']) {
            update_query('mod_your_addon', 
                ['external_id' => $response['id']], 
                ['client_id' => $clientId]
            );
            return ['success' => true, 'created' => true, 'external_id' => $response['id']];
        }
    }
    
    // Update existing record
    $response = your_addon_apiRequest('PUT', '/clients/' . $clientData['external_id'], $clientData);
    
    return ['success' => $response['success']];
}
```

## Hook Registration

```php
<?php
/**
 * Hook Functions
 * 
 * @package WHMCS\Module\Addons\YourAddon
 */

/**
 * Register hook handlers
 * 
 * @return array Hook definitions
 */
function your_addon_registerHooks()
{
    return [
        'ClientAdd'           => 'onClientAdd',
        'ClientEdit'          => 'onClientEdit',
        'AfterCalculateCartTotals' => 'onCalculateTotals',
    ];
}

/**
 * Handle new client creation
 * 
 * @param array $vars Hook variables
 */
function onClientAdd($vars)
{
    $clientId = $vars['userid'];
    
    // Initialize addon record for new client
    insert_query('mod_your_addon', [
        'client_id'  => $clientId,
        'status'     => 'inactive',
        'created_at' => 'NOW()',
    ]);
    
    logActivity("Initialized addon for client ID: {$clientId}", $clientId);
}

/**
 * Handle client edits
 * 
 * @param array $vars Hook variables
 */
function onClientEdit($vars)
{
    $clientId = $vars['userid'];
    
    // Trigger sync with external service
    your_addon_syncForClient($clientId);
}
```

## Smarty Template Example

```smarty
{extends file="clientarea.tpl"}

{block name="mainContent"}
<div class="your-addon-container">
    <div class="header">
        <h2>{$LANG.youraddon.title}</h2>
        <a href="{$modulelink}&action=settings" class="btn btn-default">
            {$LANG.youraddon.settings}
        </a>
    </div>
    
    <div class="status-panel">
        <div class="status-label">Status:</div>
        <div class="status-value status-{$status}">
            {$status|ucfirst}
        </div>
    </div>
    
    {if $records}
    <table class="table">
        <thead>
            <tr>
                <th>ID</th>
                <th>Created</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            {foreach $records as $record}
            <tr>
                <td>{$record.id}</td>
                <td>{$record.created_at|date_format:"Y-m-d H:i"}</td>
                <td>
                    <a href="{$modulelink}&action=view&id={$record.id}">View</a>
                </td>
            </tr>
            {/foreach}
        </tbody>
    </table>
    {else}
    <div class="alert alert-info">
        {$LANG.youraddon.no_records}
    </div>
    {/if}
</div>
{/block}
```

## Installation Checklist

1. Create module directory in `/modules/addons/`
2. Implement module information function
3. Define configuration fields
4. Implement activation/deactivation handlers
5. Create database tables during activation
6. Implement admin area output
7. Create client area output (if needed)
8. Register required hooks
9. Create Smarty templates
10. Add CSS and JavaScript assets
11. Implement upgrade path

## Required Functions

| Function | Required | Description |
|----------|----------|-------------|
| `your_addon_MetaData` | Yes | Module metadata |
| `your_addon_config` | Yes | Configuration fields |
| `your_addon_activate` | No | Activation handler |
| `your_addon_deactivate` | No | Deactivation handler |
| `your_addon_upgrade` | No | Version upgrade handler |
| `your_addon_output` | No | Admin area content |
| `your_addon_sidebar` | No | Sidebar content |
| `your_addon_clientarea` | No | Client area content |

## Best Practices for Addon Modules

- Always use namespaced prefixes for database tables
- Clean up all resources on deactivation
- Implement proper error handling
- Add logging throughout the module
- Use prepared statements for database queries
- Validate all user input
- Implement proper access controls
- Create upgrade paths for schema changes
- Document all configuration options

---

## Related Skills and Workflows

- `module-database-patterns` - Database design patterns
- `module-security-standards` - Security implementation
- `module-testing-strategies` - Testing approaches
- `module-versioning-guide` - Version management
- `module-hook-reference` - Hook points
- `smarty-template-reference` - Template development
- `client-area-theming` - Client area customization
