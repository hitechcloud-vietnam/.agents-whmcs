# WHMCS Addon Module Builder

## Concept

Addon modules extend WHMCS functionality with custom features. They can add pages to the admin area, client area, or both, and can hook into various WHMCS events.

## File Structure

```
/modules/addons/
├── youraddon/
│   ├── youraddon.php         # Main addon file
│   ├── version.php          # Version information
│   └── templates/
│       ├── admin/
│       │   └── index.tpl    # Admin template
│       └── client/
│           └── index.tpl    # Client template
```

## Core Addon Structure

```php
<?php
/**
 * Addon Module: Your Addon
 * Version: 1.0.0
 * Description: Custom addon for...
 * Author: Your Name
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Define addon information
 */
function youraddon_config()
{
    return [
        'name' => 'Your Addon',
        'description' => 'Description of what this addon does',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'api_key' => [
                'FriendlyName' => 'API Key',
                'Type' => 'password',
                'Size' => '50',
                'Description' => 'Enter your API key',
            ],
            'enable_feature' => [
                'FriendlyName' => 'Enable Feature',
                'Type' => 'yesno',
                'Description' => 'Enable or disable the feature',
            ],
            'dropdown_option' => [
                'FriendlyName' => 'Dropdown Option',
                'Type' => 'dropdown',
                'Options' => 'Option 1,Option 2,Option 3',
            ],
        ],
    ];
}

/**
 * Activate addon
 */
function youraddon_activate()
{
    // Create database tables
    $schema = "
        CREATE TABLE `mod_youraddon_data` (
            `id` INT(11) NOT NULL AUTO_INCREMENT,
            `client_id` INT(11) NOT NULL,
            `name` VARCHAR(255) NOT NULL,
            `created_at` DATETIME NOT NULL,
            PRIMARY KEY (`id`)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
    ";
    
    try {
        execute_query($schema);
        return [
            'status' => 'success',
            'description' => 'Your Addon activated successfully',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Could not activate: ' . $e->getMessage(),
        ];
    }
}

/**
 * Deactivate addon
 */
function youraddon_deactivate()
{
    try {
        // Drop tables or clean up
        drop_query("DROP TABLE `mod_youraddon_data`");
        
        return [
            'status' => 'success',
            'description' => 'Your Addon deactivated successfully',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Could not deactivate: ' . $e->getMessage(),
        ];
    }
}

/**
 * Upgrade addon
 */
function youraddon_upgrade($oldVersion)
{
    if (version_compare($oldVersion, '1.1.0', '<')) {
        // Run upgrade scripts
        $alter = "ALTER TABLE `mod_youraddon_data` ADD COLUMN `updated_at` DATETIME NULL";
        execute_query($alter);
    }
    
    return [
        'status' => 'success',
        'description' => 'Upgrade completed',
    ];
}

/**
 * Admin output
 */
function youraddon_output(array $params)
{
    $action = isset($_GET['action']) ? $_GET['action'] : 'index';
    
    switch ($action) {
        case 'manage':
            return youraddon_manage_page($params);
        case 'settings':
            return youraddon_settings_page($params);
        case 'export':
            return youraddon_export_data($params);
        default:
            return youraddon_index_page($params);
    }
}

function youraddon_index_page(array $params)
{
    // Fetch data
    $data = select_query('mod_youraddon_data', '*', '', 'id', 'DESC', '0,50');
    
    // Build table
    $table = new \WHMCS\Table('mod_youraddon');
    $table->setColumns([
        'ID' => ['width' => '10%'],
        'Client' => ['width' => '30%'],
        'Name' => ['width' => '30%'],
        'Created' => ['width' => '20%'],
        'Actions' => ['width' => '10%'],
    ]);
    
    while ($row = mysql_fetch_array($data)) {
        $client = Illuminate\Database\Capsule\Manager::table('tblclients')
            ->find($row['client_id']);
        
        $table->addRow([
            'ID' => $row['id'],
            'Client' => $client ? '<a href="clients.php?userid=' . $row['client_id'] . '">' 
                . htmlspecialchars($client->firstname . ' ' . $client->lastname) . '</a>' : 'N/A',
            'Name' => htmlspecialchars($row['name']),
            'Created' => fromMySQLDate($row['created_at']),
            'Actions' => '<a href="?module=youraddon&action=manage&id=' . $row['id'] 
                . '" class="btn btn-xs btn-default">Edit</a>',
        ]);
    }
    
    return [
        'breadcrumb' => [
            'index' => 'Your Addon',
        ],
        'templatefile' => 'admin/index',
        'vars' => [
            'table' => $table->output(),
            'stats' => [
                'total' => count($data),
            ],
        ],
    ];
}

function youraddon_manage_page(array $params)
{
    $id = isset($_GET['id']) ? (int)$_GET['id'] : 0;
    
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        // Handle form submission
        $name = isset($_POST['name']) ? trim($_POST['name']) : '';
        
        update_query('mod_youraddon_data', [
            'name' => $name,
            'updated_at' => date('Y-m-d H:i:s'),
        ], ['id' => $id]);
        
        redir('module=youraddon');
    }
    
    $data = select_query('mod_youraddon_data', '*', ['id' => $id]);
    $row = mysql_fetch_assoc($data);
    
    return [
        'templatefile' => 'admin/manage',
        'vars' => [
            'data' => $row,
            'id' => $id,
        ],
    ];
}

/**
 * Client area output
 */
function youraddon_clientarea(array $params)
{
    global $smarty;
    
    // Check access
    $client = Menu::context('client');
    
    return [
        'pagetitle' => 'Your Addon',
        'breadcrumb' => [
            'index' => 'Your Addon',
        ],
        'templatefile' => 'client/index',
        'vars' => [
            'client' => $client,
            'data' => youraddon_get_client_data($client->id),
        ],
    ];
}

function youraddon_get_client_data(int $clientId)
{
    return select_query('mod_youraddon_data', '*', ['client_id' => $clientId]);
}
```

## Admin Template Example

```smarty
<div class="container-fluid">
    <div class="row">
        <div class="col-md-12">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <i class="fa fa-plus"></i> Your Addon
                    <div class="pull-right">
                        <a href="{$smarty.server.PHP_SELF}?module=youraddon&action=export" 
                           class="btn btn-xs btn-default">
                            <i class="fa fa-download"></i> Export
                        </a>
                    </div>
                </div>
                <div class="panel-body">
                    {if $message}
                    <div class="alert alert-success">{$message}</div>
                    {/if}
                    
                    {$table}
                </div>
                <div class="panel-footer">
                    <a href="{$smarty.server.PHP_SELF}?module=youraddon&action=add" 
                       class="btn btn-primary">
                        Add New
                    </a>
                </div>
            </div>
        </div>
    </div>
</div>
```

## Hook Integration

```php
// Add hook within addon
function youraddon_activate()
{
    // Register hooks
    $hooks = [
        ['hook' => 'ClientAreaPrimaryNavbar', 'priority' => 1000],
        ['hook' => 'ClientAreaPage', 'priority' => 1000],
        ['hook' => 'InvoiceCreationPreTax', 'priority' => 1000],
    ];
    
    foreach ($hooks as $hook) {
        insert_query('tblhooks', [
            'hook' => $hook['hook'],
            'module' => 'youraddon',
            'priority' => $hook['priority'],
        ]);
    }
    
    return ['status' => 'success'];
}
```

## Step-by-Step Implementation

1. Create module directory in `/modules/addons/youraddon/`
2. Create main addon file with config function
3. Implement activation/deactivation functions
4. Create admin output function
5. Create client area function
6. Add upgrade path
7. Create templates
8. Add database migrations

## Implementation Checklist

- [ ] Create module directory structure
- [ ] Implement config function with fields
- [ ] Implement activation function
- [ ] Implement deactivation function
- [ ] Implement upgrade function
- [ ] Implement admin output function
- [ ] Implement client area function
- [ ] Create admin templates
- [ ] Create client templates
- [ ] Add database tables
- [ ] Register hooks
- [ ] Test activation/deactivation