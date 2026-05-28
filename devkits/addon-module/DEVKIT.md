# WHMCS Addon Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/addon-module/
├── addon.php           # Main addon module
├── templates/
│   ├── admin.tpl       # Admin area template
│   └── clientarea.tpl  # Client area template
├── lang/
│   └── english.php     # Language file
├── hooks.php           # Hooks file
└── DEVKIT.md           # This file
```

## Template

```php
<?php
/**
 * WHMCS Addon Module: {addon}
 * Addon Module Template
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function {addon}_config(): array {
    return [
        'name' => '{Module Name}',
        'description' => '{Module Description}',
        'version' => '1.0',
        'author' => '{Author Name}',
        'language' => 'english',

        // Settings
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '50',
        ],
        'api_secret' => [
            'FriendlyName' => 'API Secret',
            'Type' => 'password',
            'Size' => '50',
        ],
        'test_mode' => [
            'FriendlyName' => 'Test Mode',
            'Type' => 'yesno',
        ],
    ];
}

function {addon}_activate(): array {
    try {
        // Create main data table
        if (!Capsule::schema()->hasTable('mod_{addon}_data')) {
            Capsule::schema()->create('mod_{addon}_data', function($t) {
                $t->increments('id');
                $t->string('name', 255);
                $t->text('description')->nullable();
                $t->string('status', 50)->default('active');
                $t->timestamps();
            });
        }

        // Create settings table
        if (!Capsule::schema()->hasTable('mod_{addon}_settings')) {
            Capsule::schema()->create('mod_{addon}_settings', function($t) {
                $t->increments('id');
                $t->string('setting_key', 100)->unique();
                $t->text('setting_value')->nullable();
                $t->timestamp('updated_at');
            });
        }

        // Create logs table
        if (!Capsule::schema()->hasTable('mod_{addon}_logs')) {
            Capsule::schema()->create('mod_{addon}_logs', function($t) {
                $t->increments('id');
                $t->string('level', 20)->default('info');
                $t->text('message');
                $t->text('context')->nullable();
                $t->timestamp('created_at');
            });
        }

        return [
            'status' => 'success',
            'description' => '{Addon} module activated successfully',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Activation failed: ' . $e->getMessage(),
        ];
    }
}

function {addon}_deactivate(): array {
    try {
        Capsule::schema()->dropIfExists('mod_{addon}_data');
        Capsule::schema()->dropIfExists('mod_{addon}_settings');
        Capsule::schema()->dropIfExists('mod_{addon}_logs');

        return [
            'status' => 'success',
            'description' => '{Addon} module deactivated successfully',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Deactivation failed: ' . $e->getMessage(),
        ];
    }
}

function {addon}_upgrade(array $vars): void {
    $version = $vars['version'];

    if (version_compare($version, '1.1', '<')) {
        // Migration 1.0 -> 1.1
        Capsule::schema()->table('mod_{addon}_data', function($t) {
            if (!$t->hasColumn('new_field')) {
                $t->string('new_field', 100)->nullable();
            }
        });
    }

    if (version_compare($version, '1.2', '<')) {
        // Migration 1.1 -> 1.2
        if (!Capsule::schema()->hasTable('mod_{addon}_cache')) {
            Capsule::schema()->create('mod_{addon}_cache', function($t) {
                $t->increments('id');
                $t->string('key', 100)->unique();
                $t->text('value');
                $t->timestamp('expires_at');
            });
        }
    }
}

function {addon}_output(array $vars): void {
    // Check for CSRF on POST
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');

        $action = $_POST['action'] ?? '';
        handleAdminAction($action);
    }

    // Determine current view
    $action = $_REQUEST['action'] ?? 'dashboard';

    // Load template
    $templateDir = __DIR__ . '/templates/admin/';
    $templateFile = $templateDir . $action . '.tpl';

    if (!file_exists($templateFile)) {
        $templateFile = $templateDir . 'dashboard.tpl';
    }

    // Prepare data for template
    $templateData = prepareAdminData($action);

    // Render template
    echo '<div class="addon-container">';
    echo '<div class="addon-header">';
    echo '<h1>{Module Name}</h1>';
    echo '<a href="?module={addon}&action=settings" class="btn">Settings</a>';
    echo '</div>';
    echo '<div class="addon-content">';
    include $templateFile;
    echo '</div>';
    echo '</div>';
}

function {addon}_clientarea(array $vars): array {
    return [
        'pagetitle' => '{Module Name}',
        'templatefile' => 'templates/clientarea/clientarea',
        'vars' => [
            'user_id' => $_SESSION['uid'],
            'services' => getUserServices($_SESSION['uid']),
        ],
        'requirelogin' => true,
    ];
}

function {addon}_sidebar(array $vars): string {
    return '<div class="addon-sidebar">
        <h4>{Module}</h4>
        <ul>
            <li><a href="?module={addon}">Dashboard</a></li>
            <li><a href="?module={addon}&action=settings">Settings</a></li>
        </ul>
    </div>';
}

// Helper Functions
function handleAdminAction(string $action): void {
    switch ($action) {
        case 'create':
            createNewItem();
            break;
        case 'update':
            updateItem();
            break;
        case 'delete':
            deleteItem();
            break;
        case 'settings':
            saveSettings();
            break;
    }

    // Redirect back
    header('Location: ?module={addon}&action=' . ($_POST['redirect'] ?? 'dashboard'));
    exit;
}

function prepareAdminData(string $action): array {
    return match ($action) {
        'dashboard' => getDashboardData(),
        'data' => getDataList(),
        'logs' => getLogsList(),
        'settings' => getSettingsData(),
        default => getDashboardData(),
    };
}

function getDashboardData(): array {
    return [
        'total_items' => Capsule::table('mod_{addon}_data')->count(),
        'active_items' => Capsule::table('mod_{addon}_data')->where('status', 'active')->count(),
        'recent_logs' => Capsule::table('mod_{addon}_logs')
            ->orderBy('created_at', 'desc')
            ->limit(10)
            ->get(),
    ];
}
```

## Admin Template

```smarty
<div class="admin-addon">
    <nav class="admin-tabs">
        <a href="?module={addon}" class="{if $action eq 'dashboard'}active{/if}">Dashboard</a>
        <a href="?module={addon}&action=data" class="{if $action eq 'data'}active{/if}">Data</a>
        <a href="?module={addon}&action=logs" class="{if $action eq 'logs'}active{/if}">Logs</a>
        <a href="?module={addon}&action=settings" class="{if $action eq 'settings'}active{/if}">Settings</a>
    </nav>

    <div class="admin-panel">
        {if $action eq 'dashboard'}
        <div class="stats-grid">
            <div class="stat-box">
                <div class="stat-value">{$total_items}</div>
                <div class="stat-label">Total Items</div>
            </div>
            <div class="stat-box">
                <div class="stat-value">{$active_items}</div>
                <div class="stat-label">Active</div>
            </div>
        </div>
        {/if}

        {if $action eq 'data'}
        <div class="toolbar">
            <a href="?module={addon}&action=create" class="btn btn-primary">Add New</a>
        </div>
        <table class="data-table">
            <thead>
                <tr>
                    <th>ID</th>
                    <th>Name</th>
                    <th>Status</th>
                    <th>Created</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody>
                {foreach $items as $item}
                <tr>
                    <td>{$item.id}</td>
                    <td>{$item.name|escape:'html'}</td>
                    <td><span class="badge badge-{$item.status}">{$item.status}</span></td>
                    <td>{$item.created_at|date_format:'%Y-%m-d'}</td>
                    <td>
                        <a href="?module={addon}&action=edit&id={$item.id}" class="btn btn-sm">Edit</a>
                        <a href="?module={addon}&action=delete&id={$item.id}" class="btn btn-sm btn-danger">Delete</a>
                    </td>
                </tr>
                {/foreach}
            </tbody>
        </table>
        {/if}

        {if $action eq 'logs'}
        <table class="data-table">
            <thead>
                <tr>
                    <th>Date</th>
                    <th>Level</th>
                    <th>Message</th>
                </tr>
            </thead>
            <tbody>
                {foreach $logs as $log}
                <tr>
                    <td>{$log.created_at|date_format:'%Y-m-d H:i'}</td>
                    <td><span class="badge badge-{$log.level}">{$log.level}</span></td>
                    <td>{$log.message|escape:'html'}</td>
                </tr>
                {/foreach}
            </tbody>
        </table>
        {/if}

        {if $action eq 'settings'}
        <form method="post" action="?module={addon}&action=settings">
            <input type="hidden" name="token" value="{$token}">
            <input type="hidden" name="action" value="settings">
            <input type="hidden" name="redirect" value="settings">

            <div class="form-group">
                <label>API Key</label>
                <input type="password" name="api_key" value="{$settings.api_key}" class="form-control">
            </div>

            <button type="submit" class="btn btn-primary">Save Settings</button>
        </form>
        {/if}
    </div>
</div>
```

## Client Area Template

```smarty
<div class="client-addon">
    <h2>{Module Name}</h2>

    <div class="client-info">
        <p>Welcome to your module dashboard!</p>
    </div>

    <div class="service-list">
        {foreach $services as $service}
        <div class="service-item">
            <h3>{$service.domain}</h3>
            <p>Status: {$service.status}</p>
            <a href="?m={addon}&action=view&id={$service.id}" class="btn">
                View Details
            </a>
        </div>
        {/foreach}
    </div>
</div>
```

## Checklist

```
Pre-Dev:
□ Plan database tables (prefix with mod_)
□ Define module settings
□ Design admin UI structure
□ Define client area features

Development:
□ Implement config() with all settings
□ Implement activate() → Create tables with mod_ prefix
□ Implement deactivate() → Drop tables
□ Implement upgrade() for version migrations
□ Implement output() with CSRF check (check_token)
□ Implement clientarea() if needed
□ Add sidebar() if needed

Template:
□ Admin table with CRUD actions
□ Forms with CSRF token
□ Statistics display
□ Logs viewer
□ Settings form

Security:
□ Always use check_token() on POST
□ Sanitize inputs
□ Escape outputs
□ Validate permissions
□ Log sensitive actions

Testing:
□ Test activation creates all tables
□ Test deactivation removes all tables
□ Test upgrade migrations
□ Test all CRUD operations
□ Verify CSRF protection
□ Test client area access
```
