# WHMCS Addon Module Workflow
# Version: 1.0 | Created: 2026-05-28

---

## Overview

This workflow guides the creation of WHMCS addon modules for admin tools, client area pages, and automation hooks.

## Prerequisites

1. Read `.agents-whmcs/CLAUDE.md` (Technical Reference)
2. Read `Core_exapm_whmcs/sample-addon-module/`
3. Read `.agents/docs/WHMCS_ADDON_NOTIFICATION.md`

---

## Module Structure

```
modules/addons/{module}/
├── {module}.php              ← Main addon file
├── hooks.php                ← Hooks registration
├── lib/
│   ├── Admin/
│   │   └── Controller.php   ← Admin controller
│   └── Client/
│       └── Controller.php   ← Client controller
├── templates/
│   ├── admin/
│   │   └── index.tpl
│   └── clientarea/
│       └── dashboard.tpl
├── lang/
│   └── english.php          ← Translations
└── logo.png                 ← 80x80px logo
```

---

## Step-by-Step Development

### Step 1: Create Main Addon File

Create `{module}.php`:

```php
<?php
/**
 * {Module} WHMCS Addon Module
 * Version: 1.0.0
 * Author: HiTechCloud
 */

use WHMCS\Database\Capsule;

if (!defined("WHMCS")) {
    die("Direct access denied");
}

/**
 * Module Configuration
 */
function {module}_config(): array {
    return [
        'name' => '{Module Name}',
        'description' => '{Description of what this module does}',
        'version' => '1.0.0',
        'author' => 'HiTechCloud',
        'fields' => [
            'api_key' => [
                'FriendlyName' => 'API Key',
                'Type' => 'password',
                'Size' => '50',
                'Description' => 'Your API key for the service',
            ],
            'webhook_url' => [
                'FriendlyName' => 'Webhook URL',
                'Type' => 'text',
                'Size' => '80',
                'Description' => 'URL to receive webhook callbacks',
            ],
            'debug_mode' => [
                'FriendlyName' => 'Debug Mode',
                'Type' => 'yesno',
                'Description' => 'Enable logging for debugging',
            ],
        ],
    ];
}

/**
 * Activation - Create required tables
 */
function {module}_activate(): array {
    try {
        // Create settings table
        if (!Capsule::schema()->hasTable('mod_{module}_settings')) {
            Capsule::schema()->create('mod_{module}_settings', function($t) {
                $t->increments('id');
                $t->string('setting_key', 100)->unique();
                $t->text('setting_value')->nullable();
                $t->timestamps();
            });
        }

        // Create logs table
        if (!Capsule::schema()->hasTable('mod_{module}_logs')) {
            Capsule::schema()->create('mod_{module}_logs', function($t) {
                $t->increments('id');
                $t->string('level', 20)->default('info');
                $t->text('message');
                $t->text('context')->nullable();
                $t->timestamp('created_at')->useCurrent();
                $t->index(['level', 'created_at']);
            });
        }

        return [
            'status' => 'success',
            'description' => 'Module activated successfully',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Activation failed: ' . $e->getMessage(),
        ];
    }
}

/**
 * Deactivation - Drop tables
 */
function {module}_deactivate(): array {
    try {
        Capsule::schema()->dropIfExists('mod_{module}_logs');
        Capsule::schema()->dropIfExists('mod_{module}_settings');

        return [
            'status' => 'success',
            'description' => 'Module deactivated and tables removed',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Deactivation failed: ' . $e->getMessage(),
        ];
    }
}

/**
 * Upgrade - Handle version migrations
 */
function {module}_upgrade(array $vars): void {
    $fromVersion = $vars['version'];

    if (version_compare($fromVersion, '1.1', '<')) {
        // Migration from 1.0 to 1.1
        Capsule::schema()->table('mod_{module}_logs', function($t) {
            if (!Capsule::schema()->hasColumn('mod_{module}_logs', 'extra_data')) {
                $t->text('extra_data')->nullable();
            }
        });
    }
}

/**
 * Admin Output
 */
function {module}_output(array $vars): void {
    // Check CSRF token for POST requests
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');

        $action = $vars['action'] ?? 'default';

        switch ($action) {
            case 'save_settings':
                saveSettings($_POST);
                break;
            case 'clear_logs':
                clearLogs();
                break;
        }

        // Redirect back with success message
        header('Location: ' . $_SERVER['PHP_SELF'] . '?module={module}&success=1');
        exit;
    }

    // Get settings and logs
    $settings = getSettings();
    $logs = getLogs(50);

    // Display output
    echo '<div class="module-header">';
    echo '<h1>{Module Name}</h1>';
    echo '</div>';

    echo '<div class="module-content">';

    // Tabs
    echo '<div class="whmcs-tabs">';
    echo '<a href="?module={module}&tab=settings" class="' . ($tab === 'settings' ? 'active' : '') . '">Settings</a>';
    echo '<a href="?module={module}&tab=logs" class="' . ($tab === 'logs' ? 'active' : '') . '">Logs</a>';
    echo '<a href="?module={module}&tab=help" class="' . ($tab === 'help' ? 'active' : '') . '">Help</a>';
    echo '</div>';

    // Tab content
    echo '<div class="tab-content">';

    if ($tab === 'settings') {
        include dirname(__FILE__) . '/templates/admin/settings.tpl';
    } elseif ($tab === 'logs') {
        include dirname(__FILE__) . '/templates/admin/logs.tpl';
    } else {
        include dirname(__FILE__) . '/templates/admin/help.tpl';
    }

    echo '</div>';
    echo '</div>';
}

/**
 * Client Area Output
 */
function {module}_clientarea(array $vars): array {
    // Check if user is logged in
    if (!session_get('uid')) {
        return [
            'pagetitle' => 'Login Required',
            'templatefile' => 'login_required',
        ];
    }

    $userId = session_get('uid');

    // Handle actions
    $action = $vars['action'] ?? 'dashboard';

    switch ($action) {
        case 'dashboard':
            $data = getDashboardData($userId);
            break;
        case 'details':
            $data = getDetailsData($userId, $vars['id']);
            break;
        default:
            $data = getDashboardData($userId);
    }

    return [
        'pagetitle' => '{Module Name}',
        'templatefile' => 'clientarea/' . $action,
        'vars' => $data,
        'requirelogin' => true,
    ];
}

// Helper functions

function getSettings(): array {
    return Capsule::table('mod_{module}_settings')
        ->pluck('setting_value', 'setting_key')
        ->toArray();
}

function saveSettings(array $data): void {
    foreach ($data as $key => $value) {
        Capsule::table('mod_{module}_settings')
            ->updateOrInsert(
                ['setting_key' => $key],
                ['setting_value' => $value, 'updated_at' => date('Y-m-d H:i:s')]
            );
    }
}

function getLogs(int $limit = 100): array {
    return Capsule::table('mod_{module}_logs')
        ->orderBy('created_at', 'desc')
        ->limit($limit)
        ->get()
        ->toArray();
}

function clearLogs(): void {
    Capsule::table('mod_{module}_logs')->truncate();
}

function logModuleMessage(string $level, string $message, array $context = []): void {
    Capsule::table('mod_{module}_logs')->insert([
        'level' => $level,
        'message' => $message,
        'context' => json_encode($context),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

function getDashboardData(int $userId): array {
    return [
        'total_items' => 0,
        'recent_items' => [],
    ];
}

function getDetailsData(int $userId, int $id): array {
    return ['id' => $id];
}
```

### Step 2: Create Hooks File

Create `hooks.php`:

```php
<?php
/**
 * {Module} Hooks
 * Version: 1.0.0
 */

use WHMCS\Database\Capsule;

if (!defined("WHMCS")) {
    die("Direct access denied");
}

/**
 * Hook: Client Added
 */
add_hook('ClientAdd', 1, function($vars) {
    $userId = $vars['userid'];

    logModuleMessage('info', 'New client registered', [
        'userid' => $userId,
        'email' => $vars['email'] ?? '',
    ]);

    // Your custom logic here
});

/**
 * Hook: After Module Create (Service Created)
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $userId = $vars['userid'];

    logModuleMessage('info', 'Service created', [
        'serviceid' => $serviceId,
        'userid' => $userId,
    ]);

    // Your custom logic here
});

/**
 * Hook: Invoice Paid
 */
add_hook('InvoicePaid', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];
    $userId = $vars['userid'];
    $amount = $vars['amount'];

    logModuleMessage('info', 'Invoice paid', [
        'invoiceid' => $invoiceId,
        'userid' => $userId,
        'amount' => $amount,
    ]);

    // Your custom logic here
});

/**
 * Hook: Daily Cron Job
 */
add_hook('DailyCronJob', 1, function($vars) {
    logModuleMessage('info', 'Daily cron started');

    // Your scheduled task logic here
    // e.g., sync data, send notifications, cleanup
});
```

### Step 3: Create Language File

Create `lang/english.php`:

```php
<?php
/**
 * {Module} Language Strings
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

return [
    'module_name' => '{Module Name}',
    'tab_settings' => 'Settings',
    'tab_logs' => 'Logs',
    'tab_help' => 'Help',

    // Settings
    'settings_title' => 'Module Settings',
    'settings_api_key' => 'API Key',
    'settings_webhook_url' => 'Webhook URL',
    'settings_debug_mode' => 'Debug Mode',
    'settings_save' => 'Save Settings',

    // Logs
    'logs_title' => 'Activity Logs',
    'logs_level' => 'Level',
    'logs_message' => 'Message',
    'logs_time' => 'Time',
    'logs_clear' => 'Clear Logs',

    // Messages
    'msg_saved' => 'Settings saved successfully',
    'msg_cleared' => 'Logs cleared successfully',
    'msg_error' => 'An error occurred',

    // Client area
    'client_title' => 'Dashboard',
    'client_total' => 'Total Items',
    'client_recent' => 'Recent Activity',
];
```

### Step 4: Create Admin Template

Create `templates/admin/index.tpl`:

```html
<div class="module-settings">
    <div class="row">
        <div class="col-md-12">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <i class="fa fa-cog"></i> Settings
                </div>
                <div class="panel-body">
                    <form method="POST" action="{$smarty.server.PHP_SELF}?module={module}&action=save_settings">
                        <input type="hidden" name="token" value="{$token}">

                        <div class="form-group">
                            <label>API Key</label>
                            <input type="password" name="api_key" value="{$settings.api_key}" class="form-control">
                        </div>

                        <div class="form-group">
                            <label>Webhook URL</label>
                            <input type="text" name="webhook_url" value="{$settings.webhook_url}" class="form-control">
                        </div>

                        <div class="form-group">
                            <label>
                                <input type="checkbox" name="debug_mode" value="1" {$settings.debug_mode ? 'checked' : ''}>
                                Enable Debug Mode
                            </label>
                        </div>

                        <button type="submit" class="btn btn-primary">
                            <i class="fa fa-save"></i> Save Settings
                        </button>
                    </form>
                </div>
            </div>
        </div>
    </div>
</div>
```

---

## Checklist

- [ ] Module config with all settings
- [ ] activate() creates tables with mod_ prefix
- [ ] deactivate() drops tables
- [ ] upgrade() handles migrations
- [ ] output() with CSRF protection
- [ ] clientarea() for client-facing pages
- [ ] hooks.php for all hooks
- [ ] lang/english.php for translations
- [ ] check_token() on all POST requests

---

## Common Issues

| Issue | Solution |
|-------|----------|
| Tables not created | Check Capsule::schema() syntax |
| CSRF error | Ensure check_token() is called |
| Module not showing | Check file naming (module.php not {module}.php) |
| Client area 403 | Check requirelogin and session |

---

Last updated: 2026-05-28