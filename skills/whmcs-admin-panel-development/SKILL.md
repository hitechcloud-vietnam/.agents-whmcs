# WHMCS Admin Panel Development Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for developing custom admin panel functionality, including admin pages, dashboards, reports, and backend interfaces within WHMCS.

## When to Use

- Creating custom admin pages and dashboards
- Building admin reports and analytics
- Developing admin module interfaces
- Extending admin panel functionality
- Creating admin-specific tools and utilities

## Admin Panel Architecture

### 1. Addon Module Admin Output

```php
<?php
// modules/addons/myaddon/admin.php

if (!defined("WHMCS")) { die("Direct access denied"); }

// Determine which page to show
$action = isset($_REQUEST['action']) ? $_REQUEST['action'] : 'dashboard';

// Security: Check admin permissions
if (!function_exists('check_permission')) {
    throw new \Exception('Permission check function not available');
}

// Route to appropriate handler
switch ($action) {
    case 'dashboard':
        include __DIR__ . '/admin/dashboard.php';
        break;
    case 'clients':
        include __DIR__ . '/admin/clients.php';
        break;
    case 'reports':
        include __DIR__ . '/admin/reports.php';
        break;
    case 'settings':
        include __DIR__ . '/admin/settings.php';
        break;
    default:
        include __DIR__ . '/admin/dashboard.php';
}
```

### 2. Admin Dashboard Page

```php
<?php
// modules/addons/myaddon/admin/dashboard.php

if (!defined("WHMCS")) { die("Direct access denied"); }

use WHMCS\Database\Capsule;

// Get statistics
$stats = [
    'total_clients' => Capsule::table('tblclients')->count(),
    'active_services' => Capsule::table('tblhosting')->where('domainstatus', 'Active')->count(),
    'pending_orders' => Capsule::table('tblorders')->where('status', 'Pending')->count(),
    'overdue_invoices' => Capsule::table('tblinvoices')
        ->where('status', 'Unpaid')
        ->where('duedate', '<', date('Y-m-d'))
        ->count(),
];

// Get recent activity
$recentActivity = Capsule::table('mod_myaddon_activity')
    ->orderBy('created_at', 'DESC')
    ->limit(10)
    ->get();

// Get chart data
$monthlyData = Capsule::table('tblorders')
    ->selectRaw('DATE_FORMAT(date, "%Y-%m") as month, COUNT(*) as count, SUM(totaldue) as revenue')
    ->where('date', '>=', date('Y-m-d', strtotime('-12 months')))
    ->groupBy('month')
    ->orderBy('month')
    ->get();

$smarty = new \WHMCS\Smarty\Frontend();
$smarty->assign('stats', $stats);
$smarty->assign('recent_activity', $recentActivity);
$smarty->assign('monthly_data', $monthlyData);
$smarty->assign('page_title', 'MyAddon Dashboard');

echo $smarty->fetch(__DIR__ . '/templates/admin/dashboard.tpl');
```

### 3. Admin List View (DataTable)

```php
<?php
// modules/addons/myaddon/admin/clients.php

if (!defined("WHMCS")) { die("Direct access denied"); }

// Handle POST requests
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    check_token('WHMCS.admin.default');

    $action = $_POST['action'] ?? '';

    switch ($action) {
        case 'export':
            exportClients();
            break;
        case 'delete':
            deleteClients($_POST['ids']);
            break;
        case 'update_status':
            updateClientStatus($_POST['id'], $_POST['status']);
            break;
    }
}

// Get filter parameters
$filter = [
    'status' => $_GET['status'] ?? '',
    'search' => $_GET['search'] ?? '',
    'page' => (int) ($_GET['page'] ?? 1),
    'limit' => 25,
];

// Build query
$query = Capsule::table('mod_myaddon_clients');

if ($filter['status']) {
    $query->where('status', $filter['status']);
}

if ($filter['search']) {
    $query->where(function($q) use ($filter) {
        $q->where('name', 'like', '%' . $filter['search'] . '%')
          ->orWhere('email', 'like', '%' . $filter['search'] . '%');
    });
}

// Get total count
$total = $query->count();

// Get paginated results
$clients = $query
    ->orderBy('id', 'DESC')
    ->offset(($filter['page'] - 1) * $filter['limit'])
    ->limit($filter['limit'])
    ->get();

// Assign to template
$smarty->assign('clients', $clients);
$smarty->assign('pagination', [
    'total' => $total,
    'page' => $filter['page'],
    'limit' => $filter['limit'],
    'pages' => ceil($total / $filter['limit']),
]);
$smarty->assign('filter', $filter);

echo $smarty->fetch(__DIR__ . '/templates/admin/clients.tpl');
```

### 4. Admin Form Handler

```php
<?php
// modules/addons/myaddon/admin/settings.php

if (!defined("WHMCS")) { die("Direct access denied"); }

// Handle form submission
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    check_token('WHMCS.admin.default');

    $errors = [];

    // Validate inputs
    $apiKey = trim($_POST['api_key'] ?? '');
    if (empty($apiKey)) {
        $errors['api_key'] = 'API Key is required';
    }

    $webhookUrl = trim($_POST['webhook_url'] ?? '');
    if (!empty($webhookUrl) && !filter_var($webhookUrl, FILTER_VALIDATE_URL)) {
        $errors['webhook_url'] = 'Invalid webhook URL';
    }

    $retryCount = (int) ($_POST['retry_count'] ?? 3);
    if ($retryCount < 0 || $retryCount > 10) {
        $errors['retry_count'] = 'Retry count must be between 0 and 10';
    }

    // Save if no errors
    if (empty($errors)) {
        Capsule::table('mod_myaddon_config')->updateOrInsert(
            ['setting' => 'api_key'],
            ['value' => encrypt($apiKey), 'updated_at' => date('Y-m-d H:i:s')]
        );

        Capsule::table('mod_myaddon_config')->updateOrInsert(
            ['setting' => 'webhook_url'],
            ['value' => $webhookUrl, 'updated_at' => date('Y-m-d H:i:s')]
        );

        Capsule::table('mod_myaddon_config')->updateOrInsert(
            ['setting' => 'retry_count'],
            ['value' => $retryCount, 'updated_at' => date('Y-m-d H:i:s')]
        );

        // Handle checkbox (boolean settings)
        $enableNotifications = isset($_POST['enable_notifications']) ? 1 : 0;
        Capsule::table('mod_myaddon_config')->updateOrInsert(
            ['setting' => 'enable_notifications'],
            ['value' => $enableNotifications, 'updated_at' => date('Y-m-d H:i:s')]
        );

        // Handle multi-select
        if (isset($_POST['enabled_events']) && is_array($_POST['enabled_events'])) {
            Capsule::table('mod_myaddon_config')->updateOrInsert(
                ['setting' => 'enabled_events'],
                ['value' => json_encode($_POST['enabled_events']), 'updated_at' => date('Y-m-d H:i:s')]
            );
        }

        logActivity('MyAddon settings updated by admin');

        // Set success message
        $_SESSION['myaddon_success'] = 'Settings saved successfully';
        header('Location: ' . $_SERVER['REQUEST_URI']);
        exit;
    }
}

// Load current settings
$settings = [];
$configRows = Capsule::table('mod_myaddon_config')->get();
foreach ($configRows as $row) {
    $settings[$row->setting] = $row->setting === 'api_key'
        ? decrypt($row->value)
        : $row->value;
}

$smarty->assign('settings', $settings);
$smarty->assign('errors', $errors ?? []);
$smarty->assign('available_events', [
    'invoice_paid' => 'Invoice Paid',
    'invoice_overdue' => 'Invoice Overdue',
    'service_created' => 'Service Created',
    'service_suspended' => 'Service Suspended',
    'ticket_opened' => 'Ticket Opened',
]);

echo $smarty->fetch(__DIR__ . '/templates/admin/settings.tpl');
```

## Admin Template Structure

### 1. Admin Page Template

```html
<!-- templates/admin/clients.tpl -->

<div class="row">
    <div class="col-md-12">
        <div class="panel panel-default">
            <div class="panel-heading">
                <div class="pull-left">
                    <h3 class="panel-title">Manage Clients</h3>
                </div>
                <div class="pull-right">
                    <a href="?action=add" class="btn btn-primary btn-sm">
                        <i class="fa fa-plus"></i> Add Client
                    </a>
                    <a href="?action=export" class="btn btn-default btn-sm">
                        <i class="fa fa-download"></i> Export
                    </a>
                </div>
            </div>
            <div class="panel-body">
                <!-- Filter Form -->
                <form method="get" class="form-inline pull-right" style="margin-bottom: 15px;">
                    <input type="hidden" name="action" value="clients">
                    <div class="form-group">
                        <input type="text" name="search" class="form-control"
                               placeholder="Search..."
                               value="{$filter.search}">
                    </div>
                    <div class="form-group">
                        <select name="status" class="form-control">
                            <option value="">All Status</option>
                            <option value="active" {if $filter.status eq 'active'}selected{/if}>Active</option>
                            <option value="inactive" {if $filter.status eq 'inactive'}selected{/if}>Inactive</option>
                        </select>
                    </div>
                    <button type="submit" class="btn btn-default">Filter</button>
                </form>

                <!-- Data Table -->
                <table class="table table-striped">
                    <thead>
                        <tr>
                            <th><input type="checkbox" id="select-all"></th>
                            <th>ID</th>
                            <th>Name</th>
                            <th>Email</th>
                            <th>Status</th>
                            <th>Created</th>
                            <th>Actions</th>
                        </tr>
                    </thead>
                    <tbody>
                        {foreach $clients as $client}
                        <tr>
                            <td><input type="checkbox" name="ids[]" value="{$client.id}"></td>
                            <td>{$client.id}</td>
                            <td>{$client.name}</td>
                            <td>{$client.email}</td>
                            <td>
                                <span class="label label-{if $client.status eq 'active'}success{else}default{/if}">
                                    {$client.status}
                                </span>
                            </td>
                            <td>{$client.created_at}</td>
                            <td>
                                <a href="?action=view&id={$client.id}" class="btn btn-xs btn-default">
                                    <i class="fa fa-eye"></i>
                                </a>
                                <a href="?action=edit&id={$client.id}" class="btn btn-xs btn-primary">
                                    <i class="fa fa-edit"></i>
                                </a>
                                <a href="?action=delete&id={$client.id}"
                                   class="btn btn-xs btn-danger"
                                   onclick="return confirm('Are you sure?')">
                                    <i class="fa fa-trash"></i>
                                </a>
                            </td>
                        </tr>
                        {foreachelse}
                        <tr>
                            <td colspan="7" class="text-center">No clients found</td>
                        </tr>
                        {/foreach}
                    </tbody>
                </table>

                <!-- Pagination -->
                {if $pagination.pages > 1}
                <div class="text-center">
                    <ul class="pagination">
                        {for $i=1 to $pagination.pages}
                        <li {if $i eq $pagination.page}class="active"{/if}>
                            <a href="?action=clients&page={$i}&search={$filter.search}&status={$filter.status}">
                                {$i}
                            </a>
                        </li>
                        {/for}
                    </ul>
                </div>
                {/if}
            </div>
        </div>
    </div>
</div>
```

### 2. Settings Form Template

```html
<!-- templates/admin/settings.tpl -->

<div class="row">
    <div class="col-md-8 col-md-offset-2">
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">MyAddon Settings</h3>
            </div>
            <div class="panel-body">
                {if $errors}
                <div class="alert alert-danger">
                    <ul class="list-unstyled">
                        {foreach $errors as $field => $message}
                        <li>{$message}</li>
                        {/foreach}
                    </ul>
                </div>
                {/if}

                <form method="post" class="form-horizontal">
                    <input type="hidden" name="token" value="{$token}">

                    <!-- API Configuration -->
                    <div class="form-group">
                        <label class="col-sm-3 control-label">API Key</label>
                        <div class="col-sm-9">
                            <input type="password" name="api_key" class="form-control"
                                   value="{$settings.api_key|default:''}"
                                   placeholder="Enter your API key">
                            <span class="help-block">Your API key for external service</span>
                        </div>
                    </div>

                    <!-- Webhook URL -->
                    <div class="form-group">
                        <label class="col-sm-3 control-label">Webhook URL</label>
                        <div class="col-sm-9">
                            <input type="url" name="webhook_url" class="form-control"
                                   value="{$settings.webhook_url|default:''}"
                                   placeholder="https://example.com/webhook">
                        </div>
                    </div>

                    <!-- Number Input -->
                    <div class="form-group">
                        <label class="col-sm-3 control-label">Retry Count</label>
                        <div class="col-sm-3">
                            <input type="number" name="retry_count" class="form-control"
                                   value="{$settings.retry_count|default:3}"
                                   min="0" max="10">
                        </div>
                    </div>

                    <!-- Checkbox -->
                    <div class="form-group">
                        <div class="col-sm-offset-3 col-sm-9">
                            <div class="checkbox">
                                <label>
                                    <input type="checkbox" name="enable_notifications" value="1"
                                           {if $settings.enable_notifications eq 1}checked{/if}>
                                    Enable notifications
                                </label>
                            </div>
                        </div>
                    </div>

                    <!-- Multi-select -->
                    <div class="form-group">
                        <label class="col-sm-3 control-label">Enabled Events</label>
                        <div class="col-sm-9">
                            <select name="enabled_events[]" class="form-control" multiple size="5">
                                {foreach $available_events as $value => $label}
                                <option value="{$value}"
                                        {if $value|in_array:$settings.enabled_events}selected{/if}>
                                    {$label}
                                </option>
                                {/foreach}
                            </select>
                            <span class="help-block">Hold Ctrl/Cmd to select multiple</span>
                        </div>
                    </div>

                    <!-- Buttons -->
                    <div class="form-group">
                        <div class="col-sm-offset-3 col-sm-9">
                            <button type="submit" class="btn btn-primary">
                                <i class="fa fa-save"></i> Save Settings
                            </button>
                            <a href="?action=reset" class="btn btn-default">
                                Reset to Defaults
                            </a>
                        </div>
                    </div>
                </form>
            </div>
        </div>
    </div>
</div>
```

## Admin API Endpoints

```php
<?php
// modules/addons/myaddon/admin/api.php

if (!defined("WHMCS")) { die("Direct access denied"); }

header('Content-Type: application/json');

// Get request data
$method = $_SERVER['REQUEST_METHOD'];
$input = json_decode(file_get_contents('php://input'), true);
$action = $_REQUEST['action'] ?? '';

// Admin authentication check
if (!isset($_SESSION['adminid'])) {
    http_response_code(401);
    echo json_encode(['error' => 'Unauthorized']);
    exit;
}

// Route handling
try {
    $response = match($action) {
        'get_clients' => handleGetClients($input),
        'get_client' => handleGetClient($input),
        'update_client' => handleUpdateClient($input),
        'delete_client' => handleDeleteClient($input),
        'get_stats' => handleGetStats($input),
        'export_data' => handleExportData($input),
        default => throw new \Exception('Unknown action'),
    };

    echo json_encode($response);

} catch (\Exception $e) {
    http_response_code(400);
    echo json_encode(['error' => $e->getMessage()]);
}

// API Handlers
function handleGetClients(array $input): array {
    $page = (int) ($input['page'] ?? 1);
    $limit = min((int) ($input['limit'] ?? 25), 100);

    $query = Capsule::table('mod_myaddon_clients');

    if (!empty($input['search'])) {
        $query->where('name', 'like', '%' . $input['search'] . '%');
    }

    $total = $query->count();
    $clients = $query->offset(($page - 1) * $limit)->limit($limit)->get();

    return [
        'success' => true,
        'data' => $clients,
        'meta' => [
            'page' => $page,
            'limit' => $limit,
            'total' => $total,
            'pages' => ceil($total / $limit),
        ],
    ];
}

function handleGetClient(array $input): array {
    $id = (int) ($input['id'] ?? 0);

    $client = Capsule::table('mod_myaddon_clients')->find($id);

    if (!$client) {
        throw new \Exception('Client not found');
    }

    return [
        'success' => true,
        'data' => $client,
    ];
}

function handleUpdateClient(array $input): array {
    check_token('WHMCS.admin.default');

    $id = (int) ($input['id'] ?? 0);
    $data = $input['data'] ?? [];

    Capsule::table('mod_myaddon_clients')
        ->where('id', $id)
        ->update($data);

    logActivity("MyAddon client {$id} updated via API");

    return [
        'success' => true,
        'message' => 'Client updated',
    ];
}

function handleDeleteClient(array $input): array {
    check_token('WHMCS.admin.default');

    $id = (int) ($input['id'] ?? 0);

    Capsule::table('mod_myaddon_clients')
        ->where('id', $id)
        ->delete();

    logActivity("MyAddon client {$id} deleted via API");

    return [
        'success' => true,
        'message' => 'Client deleted',
    ];
}

function handleGetStats(array $input): array {
    return [
        'success' => true,
        'data' => [
            'total_clients' => Capsule::table('mod_myaddon_clients')->count(),
            'active_clients' => Capsule::table('mod_myaddon_clients')->where('status', 'active')->count(),
            'monthly_revenue' => 1234.56,
        ],
    ];
}

function handleExportData(array $input): array {
    $format = $input['format'] ?? 'csv';

    $clients = Capsule::table('mod_myaddon_clients')->get();

    if ($format === 'csv') {
        header('Content-Type: text/csv');
        header('Content-Disposition: attachment; filename="clients_export.csv"');

        $output = fopen('php://output', 'w');
        fputcsv($output, ['ID', 'Name', 'Email', 'Status', 'Created']);

        foreach ($clients as $client) {
            fputcsv($output, [
                $client->id,
                $client->name,
                $client->email,
                $client->status,
                $client->created_at,
            ]);
        }

        fclose($output);
        exit;
    }

    return [
        'success' => true,
        'data' => $clients,
    ];
}
```

## Admin Permissions System

```php
<?php
// Permission checking in admin pages

// Check specific permission
function requirePermission(string $permission): void {
    if (!current_admin_has_permission($permission)) {
        // Redirect to access denied or show error
        header('Location: ?error=access_denied');
        exit;
    }
}

function current_admin_has_permission(string $permission): bool {
    $adminId = $_SESSION['adminid'] ?? 0;

    // Get admin role
    $admin = Capsule::table('tbladmins')->find($adminId);
    if (!$admin) {
        return false;
    }

    // Get role permissions
    $role = Capsule::table('tbladminroles')->find($admin->roleid);
    if (!$role) {
        return false;
    }

    $permissions = json_decode($role->permissions, true) ?? [];

    // Check specific permission
    return in_array($permission, $permissions);
}

// Permission definitions
$permissions = [
    'myaddon_view' => 'View MyAddon',
    'myaddon_manage' => 'Manage MyAddon',
    'myaddon_settings' => 'Configure Settings',
    'myaddon_export' => 'Export Data',
    'myaddon_delete' => 'Delete Records',
];

// Apply permission check
add_hook('AdminAreaPage', 1, function($vars) {
    if (strpos($vars['request_uri'], '/myaddon/') !== false) {
        requirePermission('myaddon_view');
    }
});
```

## Admin Widget System

```php
<?php
// Add widget to admin dashboard

add_hook('AdminHomeView', 1, function($vars) {
    // Only show on main dashboard
    if ($vars['filename'] !== 'index.php' || $vars['action'] !== '') {
        return;
    }

    // Get widget data
    $stats = [
        'active_services' => Capsule::table('tblhosting')->where('domainstatus', 'Active')->count(),
        'pending_orders' => Capsule::table('tblorders')->where('status', 'Pending')->count(),
    ];

    return [
        'module' => 'myaddon',
        'title' => 'MyAddon Summary',
        'template' => 'widget_summary',
        'data' => $stats,
        'options' => [
            'columns' => 2,
            'row' => 1,
        ],
    ];
});
```

## Checklist

- [ ] Admin pages use check_token() for POST requests
- [ ] CSRF protection implemented
- [ ] Permission checking for sensitive actions
- [ ] Input validation and sanitization
- [ ] Pagination for large data sets
- [ ] Proper error handling with user-friendly messages
- [ ] Activity logging for admin actions
- [ ] Responsive admin templates
- [ ] Form validation feedback

---

**Related Skills:**
- whmcs-admin-ui-builder
- whmcs-ajax-patterns
- whmcs-reporting
- whmcs-addon-builder
- whmcs-security-hardening

**Reference:**
- WHMCS Admin Development: https://developers.whmcs.com/admin-area/