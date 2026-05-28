# WHMCS Addon Module Builder Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building WHMCS addon modules for admin tools, client area pages, and automation.

## When to Use

- Creating admin management tools
- Building client portal pages
- Adding custom automation hooks
- Integrating third-party services

## Addon Structure

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
│   │   ├── index.tpl
│   │   ├── settings.tpl
│   │   └── reports.tpl
│   └── clientarea/
│       └── dashboard.tpl
├── lang/
│   └── english.php
├── css/
│   └── style.css
└── js/
    └── script.js
```

## Building Steps

### Step 1: Create Main Addon File

```php
<?php
// modules/addons/{module}/{module}.php
use WHMCS\Database\Capsule;

if (!defined("WHMCS")) { die("Direct access denied"); }

function {module}_config(): array {
    return [
        'name' => '{Module Name}',
        'description' => '{Description}',
        'version' => '1.0.0',
        'author' => 'HiTechCloud',
        'fields' => [
            'api_key' => [
                'FriendlyName' => 'API Key',
                'Type' => 'password',
                'Size' => '50',
            ],
            'webhook_url' => [
                'FriendlyName' => 'Webhook URL',
                'Type' => 'text',
                'Size' => '80',
            ],
            'debug_mode' => [
                'FriendlyName' => 'Debug Mode',
                'Type' => 'yesno',
            ],
        ],
    ];
}

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

        // Create data table
        if (!Capsule::schema()->hasTable('mod_{module}_data')) {
            Capsule::schema()->create('mod_{module}_data', function($t) {
                $t->increments('id');
                $t->integer('user_id')->unsigned();
                $t->string('type', 50);
                $t->text('data')->nullable();
                $t->timestamps();
                $t->index(['user_id', 'type']);
            });
        }

        return ['status' => 'success', 'description' => 'Activated successfully'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function {module}_deactivate(): array {
    try {
        Capsule::schema()->dropIfExists('mod_{module}_data');
        Capsule::schema()->dropIfExists('mod_{module}_logs');
        Capsule::schema()->dropIfExists('mod_{module}_settings');

        return ['status' => 'success', 'description' => 'Deactivated successfully'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function {module}_upgrade(array $vars): void {
    $fromVersion = $vars['version'];

    if (version_compare($fromVersion, '1.1', '<')) {
        Capsule::schema()->table('mod_{module}_logs', function($t) {
            if (!Capsule::schema()->hasColumn('mod_{module}_logs', 'extra_data')) {
                $t->text('extra_data')->nullable();
            }
        });
    }

    if (version_compare($fromVersion, '1.2', '<')) {
        Capsule::schema()->table('mod_{module}_data', function($t) {
            if (!Capsule::schema()->hasColumn('mod_{module}_data', 'metadata')) {
                $t->text('metadata')->nullable();
            }
        });
    }
}

function {module}_output(array $vars): void {
    // CSRF protection for POST
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');

        $action = $vars['action'] ?? $_POST['action'] ?? 'default';

        switch ($action) {
            case 'save_settings':
                saveSettings($_POST);
                break;
            case 'clear_logs':
                clearLogs();
                break;
            case 'sync_data':
                syncData();
                break;
        }

        header('Location: ' . $_SERVER['PHP_SELF'] . '?module={module}&success=1');
        exit;
    }

    $tab = $_GET['tab'] ?? 'dashboard';
    $settings = getSettings();
    $stats = getStats();

    echo '<div class="addon-container">';
    echo '<div class="addon-header"><h2>{Module Name}</h2></div>';

    echo '<div class="whmcs-tabs">';
    echo '<a href="?module={module}&tab=dashboard" class="' . ($tab === 'dashboard' ? 'active' : '') . '">Dashboard</a>';
    echo '<a href="?module={module}&tab=settings" class="' . ($tab === 'settings' ? 'active' : '') . '">Settings</a>';
    echo '<a href="?module={module}&tab=logs" class="' . ($tab === 'logs' ? 'active' : '') . '">Logs</a>';
    echo '<a href="?module={module}&tab=reports" class="' . ($tab === 'reports' ? 'active' : '') . '">Reports</a>';
    echo '</div>';

    echo '<div class="tab-content">';

    switch ($tab) {
        case 'dashboard':
            include dirname(__FILE__) . '/templates/admin/dashboard.tpl';
            break;
        case 'settings':
            include dirname(__FILE__) . '/templates/admin/settings.tpl';
            break;
        case 'logs':
            include dirname(__FILE__) . '/templates/admin/logs.tpl';
            break;
        case 'reports':
            include dirname(__FILE__) . '/templates/admin/reports.tpl';
            break;
    }

    echo '</div></div>';
}

function {module}_clientarea(array $vars): array {
    if (!session_get('uid')) {
        return ['pagetitle' => 'Login Required', 'templatefile' => 'login_required'];
    }

    $action = $vars['action'] ?? 'dashboard';
    $userId = session_get('uid');

    $data = match ($action) {
        'dashboard' => getClientDashboard($userId),
        'details' => getClientDetails($userId, $vars['id']),
        'settings' => getClientSettings($userId),
        default => getClientDashboard($userId),
    };

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
        if (in_array($key, ['api_key', 'webhook_url', 'debug_mode'])) {
            Capsule::table('mod_{module}_settings')
                ->updateOrInsert(
                    ['setting_key' => $key],
                    ['setting_value' => $value, 'updated_at' => date('Y-m-d H:i:s')]
                );
        }
    }
}

function getStats(): array {
    return [
        'total_logs' => Capsule::table('mod_{module}_logs')->count(),
        'total_data' => Capsule::table('mod_{module}_data')->count(),
    ];
}

function clearLogs(): void {
    Capsule::table('mod_{module}_logs')->truncate();
}

function getLogs(int $limit = 100): array {
    return Capsule::table('mod_{module}_logs')
        ->orderBy('created_at', 'desc')
        ->limit($limit)
        ->get()
        ->toArray();
}

function logMessage(string $level, string $message, array $context = []): void {
    Capsule::table('mod_{module}_logs')->insert([
        'level' => $level,
        'message' => $message,
        'context' => json_encode($context),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

function getClientDashboard(int $userId): array {
    return [
        'items' => Capsule::table('mod_{module}_data')
            ->where('user_id', $userId)
            ->orderBy('id', 'desc')
            ->limit(10)
            ->get()
            ->toArray(),
    ];
}

function getClientDetails(int $userId, int $id): array {
    return Capsule::table('mod_{module}_data')
        ->where('id', $id)
        ->where('user_id', $userId)
        ->first() ?? [];
}

function getClientSettings(int $userId): array {
    return ['configured' => true];
}

function syncData(): void {
    // Custom sync logic
    logMessage('info', 'Data sync completed');
}
```

### Step 2: Create Hooks File

```php
<?php
// hooks.php
if (!defined("WHMCS")) { die("Direct access denied"); }

add_hook('ClientAdd', 1, function($vars) {
    logMessage('info', 'New client', [
        'userid' => $vars['userid'],
        'email' => $vars['email'] ?? '',
    ]);
});

add_hook('AfterModuleCreate', 1, function($vars) {
    logMessage('info', 'Service created', [
        'serviceid' => $vars['serviceid'],
        'userid' => $vars['userid'],
    ]);
});

add_hook('InvoicePaid', 1, function($vars) {
    logMessage('info', 'Invoice paid', [
        'invoiceid' => $vars['invoiceid'],
        'amount' => $vars['amount'],
    ]);
});

add_hook('DailyCronJob', 1, function($vars) {
    logMessage('info', 'Daily cron started');
});
```

### Step 3: Create Language File

```php
<?php
// lang/english.php
if (!defined("WHMCS")) { die("Direct access denied"); }

return [
    'module_name' => '{Module Name}',
    'tab_dashboard' => 'Dashboard',
    'tab_settings' => 'Settings',
    'tab_logs' => 'Logs',
    'tab_reports' => 'Reports',

    'settings_title' => 'Configuration',
    'settings_api_key' => 'API Key',
    'settings_webhook' => 'Webhook URL',
    'settings_debug' => 'Debug Mode',
    'settings_save' => 'Save Settings',

    'logs_title' => 'Activity Logs',
    'logs_level' => 'Level',
    'logs_message' => 'Message',
    'logs_time' => 'Time',
    'logs_clear' => 'Clear Logs',

    'dashboard_title' => 'Dashboard',
    'dashboard_total' => 'Total Records',
    'dashboard_recent' => 'Recent Activity',

    'msg_saved' => 'Settings saved successfully',
    'msg_cleared' => 'Logs cleared successfully',
    'msg_error' => 'An error occurred',
];
```

## Checklist

- [ ] config() with all fields
- [ ] activate() creates tables with mod_ prefix
- [ ] deactivate() drops tables
- [ ] upgrade() handles migrations
- [ ] output() with CSRF check
- [ ] clientarea() with requirelogin
- [ ] hooks.php for automation
- [ ] lang/english.php for translations

---

**Related Skills:**
- whmcs-addon-database
- whmcs-addon-hooks
- whmcs-addon-clientarea