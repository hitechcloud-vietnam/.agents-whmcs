# WHMCS Addon Module Development

## Overview

Addon modules extend WHMCS functionality with custom features in both admin and client areas.

## Module Structure

```
modules/addons/youraddon/
├── youraddon.php           # Main module file
├── hooks.php               # Hooks file
├── includes/
│   └── YourClass.php       # Helper classes
├── templates/
│   ├── admin.tpl           # Admin template
│   └── client.tpl          # Client template
└── assets/
    ├── css/
    └── js/
```

## Main Module File

```php
<?php
// modules/addons/youraddon/youraddon.php

if (!defined('WHMCS')) {
    die('This file cannot be accessed directly');
}

function youraddon_config(): array
{
    return [
        'name' => 'Your Addon Name',
        'description' => 'Description of what this addon does',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'language' => 'english',
        'fields' => [
            'apiKey' => [
                'FriendlyName' => 'API Key',
                'Type' => 'password',
                'Size' => '40',
            ],
            'debugMode' => [
                'FriendlyName' => 'Enable Debug Mode',
                'Type' => 'yesno',
            ],
        ],
    ];
}

function youraddon_activate(): array
{
    try {
        // Create tables
        Capsule::schema()->create('mod_youraddon_data', function ($t) {
            $t->increments('id');
            $t->string('title');
            $t->text('content');
            $t->integer('user_id');
            $t->timestamps();
        });
        
        // Create settings table
        Capsule::schema()->create('mod_youraddon_settings', function ($t) {
            $t->increments('id');
            $t->string('setting_name');
            $t->text('setting_value');
            $t->timestamps();
        });
        
        return [
            'status' => 'success',
            'description' => 'Module activated successfully',
        ];
        
    } catch (Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed to activate: ' . $e->getMessage(),
        ];
    }
}

function youraddon_deactivate(): array
{
    try {
        Capsule::schema()->dropIfExists('mod_youraddon_data');
        Capsule::schema()->dropIfExists('mod_youraddon_settings');
        
        return [
            'status' => 'success',
            'description' => 'Module deactivated and cleaned up',
        ];
        
    } catch (Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed to deactivate: ' . $e->getMessage(),
        ];
    }
}

function youraddon_upgrade(array $vars): void
{
    $currentVersion = $vars['version'];
    
    if (version_compare($currentVersion, '1.1.0', '<')) {
        // Add new column in 1.1.0
        if (!Capsule::schema()->hasColumn('mod_youraddon_data', 'new_column')) {
            Capsule::schema()->table('mod_youraddon_data', function ($t) {
                $t->string('new_column')->nullable();
            });
        }
    }
    
    if (version_compare($currentVersion, '1.2.0', '<')) {
        // Add index in 1.2.0
        Capsule::statement('ALTER TABLE mod_youraddon_data ADD INDEX idx_user (user_id)');
    }
}

function youraddon_output(array $vars): void
{
    // Check for CSRF on POST
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
        
        $action = $_POST['action'] ?? '';
        
        switch ($action) {
            case 'save':
                youraddon_saveSettings($_POST);
                break;
            case 'delete':
                youraddon_deleteItem((int) $_POST['id']);
                break;
        }
        
        redir('module=youraddon&success=1');
    }
    
    // Get data for display
    $items = Capsule::table('mod_youraddon_data')
        ->orderBy('id', 'desc')
        ->limit(100)
        ->get();
    
    // Render output
    echo '<div class="module-header">';
    echo '<h2>Your Addon Module</h2>';
    echo '</div>';
    
    if (isset($_GET['success'])) {
        echo '<div class="alert alert-success">Action completed successfully!</div>';
    }
    
    echo '<table class="table table-striped">';
    echo '<thead><tr><th>ID</th><th>Title</th><th>Created</th><th>Actions</th></tr></thead>';
    echo '<tbody>';
    
    foreach ($items as $item) {
        echo '<tr>';
        echo '<td>' . $item->id . '</td>';
        echo '<td>' . htmlspecialchars($item->title) . '</td>';
        echo '<td>' . $item->created_at . '</td>';
        echo '<td>';
        echo '<a href="?module=youraddon&action=edit&id=' . $item->id . '" class="btn btn-sm btn-default">Edit</a> ';
        echo '<form method="post" style="display:inline;">';
        echo '<input type="hidden" name="action" value="delete">';
        echo '<input type="hidden" name="id" value="' . $item->id . '">';
        echo '<button type="submit" class="btn btn-sm btn-danger" onclick="return confirm(\'Delete?\')">Delete</button>';
        echo '</form>';
        echo '</td>';
        echo '</tr>';
    }
    
    echo '</tbody></table>';
}

function youraddon_clientarea(array $vars): array
{
    global $smarty;
    
    $userId = (int) $_SESSION['uid'];
    
    $items = Capsule::table('mod_youraddon_data')
        ->where('user_id', $userId)
        ->orderBy('id', 'desc')
        ->get();
    
    return [
        'pagetitle' => 'My Addon Page',
        'templatefile' => 'client',
        'vars' => [
            'items' => $items,
            'totalItems' => count($items),
        ],
    ];
}

// Helper functions
function youraddon_saveSettings(array $data): void
{
    foreach ($data as $key => $value) {
        if (strpos($key, 'setting_') === 0) {
            $settingName = str_replace('setting_', '', $key);
            Capsule::table('mod_youraddon_settings')
                ->updateOrInsert(
                    ['setting_name' => $settingName],
                    ['setting_value' => $value, 'updated_at' => date('Y-m-d H:i:s')]
                );
        }
    }
}

function youraddon_deleteItem(int $id): void
{
    Capsule::table('mod_youraddon_data')
        ->where('id', $id)
        ->delete();
}
```

## Hooks Integration

```php
<?php
// modules/addons/youraddon/hooks.php

add_hook('ClientCreated', 1, function($vars) {
    // Initialize addon data for new client
    Capsule::table('mod_youraddon_data')->insert([
        'user_id' => $vars['client_id'],
        'title' => 'Welcome',
        'content' => 'Welcome to our service!',
        'created_at' => date('Y-m-d H:i:s'),
        'updated_at' => date('Y-m-d H:i:s'),
    ]);
});

add_hook('InvoicePaid', 1, function($vars) {
    // Award bonus based on payment
    $userId = Capsule::table('tblinvoices')
        ->where('id', $vars['invoice_id'])
        ->value('userid');
    
    // Add to your addon data
    Capsule::table('mod_youraddon_data')->insert([
        'user_id' => $userId,
        'title' => 'Payment Bonus',
        'content' => 'Thanks for your payment!',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
});
```

## Client Template

```smarty
{include file="$templatefile/header.tpl"}

<div class="addon-container">
    <h2>My Items ({$totalItems})</h2>
    
    <div class="items-list">
        {foreach from=$items item=item}
            <div class="item-card">
                <h3>{$item->title}</h3>
                <p>{$item->content}</p>
                <small>Created: {$item->created_at}</small>
            </div>
        {foreachelse}
            <p class="no-items">No items found.</p>
        {/foreach}
    </div>
</div>

{include file="$templatefile/footer.tpl"}
```

## Admin Views

```php
<?php
function youraddon_output(array $vars): void
{
    // Get module parameters
    $apiKey = $vars['apiKey'] ?? '';
    $debugMode = $vars['debugMode'] ?? '';
    
    // Get data
    $data = getAddonData();
    
    // Include CSS
    echo '<link rel="stylesheet" href="../modules/addons/youraddon/assets/css/style.css">';
    
    // Output HTML
    echo '<div class="youraddon-wrapper">';
    echo '<!-- Your module content -->';
    echo '</div>';
    
    // Include JS
    echo '<script src="../modules/addons/youraddon/assets/js/script.js"></script>';
    echo '<script>YourAddon.init();</script>';
}
```

## Best Practices

1. **Use mod_ prefix** - All addon tables must use mod_ prefix
2. **Handle CSRF** - Always validate tokens on POST requests
3. **Return proper status** - Return `['status' => 'success']` or `['status' => 'error']`
4. **Clean up on deactivate** - Remove all tables and data
5. **Version migrations** - Handle upgrades properly

## Related Documentation

- [WHMCS Module API](/docs/whmcs-module-api.md)
- [WHMCS Hook System](/docs/whmcs-hook-system.md)