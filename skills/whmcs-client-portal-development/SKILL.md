# WHMCS Client Portal Development Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for customizing and extending the WHMCS client portal (client area), including client-facing pages, forms, dashboards, and API endpoints.

## When to Use

- Customizing client area pages
- Building client portal features
- Extending client functionality
- Creating client-specific dashboards
- Implementing client portal APIs

## Client Portal Architecture

### 1. Client Area Module Structure

```php
<?php
// modules/servers/myprovider/clientarea.php

/**
 * Client Area Page for Provisioning Module
 * Called when client visits their service details page
 */
function myprovider_ClientArea(array $params): array {
    $serviceId = $params['serviceid'];
    $clientId = $params['userid'];

    // Get service details
    $service = \WHMCS\Database\Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();

    // Get custom module data
    $moduleData = \WHMCS\Database\Capsule::table('mod_myprovider_data')
        ->where('service_id', $serviceId)
        ->first();

    // Get resource usage
    $usage = fetchResourceUsage($service->username);

    return [
        'pagetitle' => 'MyProvider Service Management',
        'templatefile' => 'clientarea',
        'vars' => [
            'service' => $service,
            'module_data' => $moduleData,
            'usage' => $usage,
            'server_status' => checkServerStatus($params),
            'quick_actions' => getQuickActions($service),
        ],
    ];
}

/**
 * Client Area Tab Pages
 */
function myprovider_ClientAreaCustomFields(array $params): array {
    return [
        'rawdata' => getModuleData($params['serviceid']),
    ];
}
```

### 2. Client Area Template

```php
<?php
// modules/servers/myprovider/templates/clientarea.tpl

<div class="client-area-module">
    <!-- Status Banner -->
    <div class="alert alert-{if $server_status eq 'online'}success{else}warning{/if}">
        <i class="fa fa-{if $server_status eq 'online'}check-circle{else}exclamation-triangle{/if}"></i>
        Server Status: <strong>{$server_status|upper}</strong>
    </div>

    <!-- Quick Actions -->
    <div class="row">
        <div class="col-sm-3">
            <a href="?action=service_management&do=restart"
               class="btn btn-block btn-default">
                <i class="fa fa-refresh"></i> Restart Server
            </a>
        </div>
        <div class="col-sm-3">
            <a href="?action=service_management&do=reboot"
               class="btn btn-block btn-default">
                <i class="fa fa-power-off"></i> Reboot
            </a>
        </div>
        <div class="col-sm-3">
            <a href="clientarea.php?action=productdetails&id={$service.id}&view=tab"
               class="btn btn-block btn-default">
                <i class="fa fa-cog"></i> Settings
            </a>
        </div>
        <div class="col-sm-3">
            <a href="supporttickets.php?action=open&related_service={$service.id}"
               class="btn btn-block btn-default">
                <i class="fa fa-ticket"></i> Support
            </a>
        </div>
    </div>

    <!-- Resource Usage -->
    <h4>Resource Usage</h4>
    <div class="row">
        <div class="col-md-4">
            <div class="usage-widget">
                <h5>CPU</h5>
                <div class="progress">
                    <div class="progress-bar" role="progressbar"
                         style="width: {$usage.cpu}%">
                        {$usage.cpu}%
                    </div>
                </div>
            </div>
        </div>
        <div class="col-md-4">
            <div class="usage-widget">
                <h5>RAM</h5>
                <div class="progress">
                    <div class="progress-bar progress-bar-info" role="progressbar"
                         style="width: {$usage.ram}%">
                        {$usage.ram}%
                    </div>
                </div>
            </div>
        </div>
        <div class="col-md-4">
            <div class="usage-widget">
                <h5>Disk</h5>
                <div class="progress">
                    <div class="progress-bar progress-bar-warning" role="progressbar"
                         style="width: {$usage.disk}%">
                        {$usage.disk}%
                    </div>
                </div>
            </div>
        </div>
    </div>

    <!-- Service Details -->
    <h4>Service Information</h4>
    <table class="table table-striped">
        <tr>
            <td width="200">Server IP</td>
            <td><strong>{$module_data.server_ip}</strong></td>
        </tr>
        <tr>
            <td>Hostname</td>
            <td>{$module_data.hostname}</td>
        </tr>
        <tr>
            <td>Operating System</td>
            <td>{$module_data.os}</td>
        </tr>
        <tr>
            <td>Control Panel</td>
            <td>{$module_data.control_panel}</td>
        </tr>
        <tr>
            <td>Next Billing Date</td>
            <td>{$service.nextduedate}</td>
        </tr>
    </table>

    <!-- Server Actions Form -->
    <form method="post" action="clientarea.php" class="form-inline">
        <input type="hidden" name="action" value="service_management">
        <input type="hidden" name="service_id" value="{$service.id}">
        <input type="hidden" name="token" value="{$token}">

        <h4>Server Control</h4>
        <div class="form-group">
            <select name="command" class="form-control">
                <option value="status">Get Status</option>
                <option value="restart">Restart Services</option>
                <option value="reinstall">Reinstall OS</option>
                <option value="rescue">Enter Rescue Mode</option>
            </select>
        </div>
        <button type="submit" class="btn btn-primary">
            Execute
        </button>
    </form>
</div>
```

## Advanced Client Portal Features

### 1. Client Portal API

```php
<?php
// modules/addons/myaddon/clientapi.php

if (!defined("WHMCS")) { die("Direct access denied"); }

/**
 * Client Portal REST API
 * Accessible at: /modules/addons/myaddon/clientapi.php
 */

use WHMCS\Database\Capsule;

// Verify client authentication
$clientId = $_SESSION['uid'] ?? 0;
if (!$clientId) {
    http_response_code(401);
    header('Content-Type: application/json');
    echo json_encode(['error' => 'Authentication required']);
    exit;
}

// Get request method and parameters
$method = $_SERVER['REQUEST_METHOD'];
$path = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
$pathParts = explode('/', trim($path, '/'));
$endpoint = end($pathParts);
$input = json_decode(file_get_contents('php://input'), true) ?: [];

// Route handling
try {
    $response = match($endpoint) {
        'dashboard' => getDashboard($clientId),
        'services' => getServices($clientId, $input),
        'service' => getServiceDetails($clientId, $input['id']),
        'invoices' => getInvoices($clientId, $input),
        'tickets' => getTickets($clientId, $input),
        'update_profile' => updateProfile($clientId, $input),
        'change_password' => changePassword($clientId, $input),
        'api_keys' => manageApiKeys($clientId, $input),
        default => throw new \Exception('Endpoint not found'),
    };

    http_response_code(200);
    header('Content-Type: application/json');
    echo json_encode($response);

} catch (\Exception $e) {
    http_response_code(400);
    header('Content-Type: application/json');
    echo json_encode(['error' => $e->getMessage()]);
}

function getDashboard(int $clientId): array {
    $client = Capsule::table('tblclients')->find($clientId);

    // Get service summary
    $services = Capsule::table('tblhosting')
        ->where('userid', $clientId)
        ->where('domainstatus', 'Active')
        ->selectRaw('COUNT(*) as total, SUM(billingcycle = "Monthly") as monthly')
        ->first();

    // Get invoice summary
    $invoices = Capsule::table('tblinvoices')
        ->where('userid', $clientId)
        ->selectRaw('SUM(status = "Unpaid") as pending, SUM(status = "Paid") as paid')
        ->first();

    // Get recent tickets
    $tickets = Capsule::table('tbltickets')
        ->where('userid', $clientId)
        ->orderBy('created', 'DESC')
        ->limit(5)
        ->get();

    return [
        'client' => [
            'name' => $client->firstname . ' ' . $client->lastname,
            'email' => $client->email,
        ],
        'summary' => [
            'active_services' => $services->total,
            'monthly_services' => $services->monthly,
            'pending_invoices' => $invoices->pending,
            'total_paid_invoices' => $invoices->paid,
        ],
        'recent_tickets' => array_map(function($t) {
            return [
                'id' => $t->id,
                'subject' => $t->title,
                'status' => $t->status,
            ];
        }, (array) $tickets),
    ];
}

function getServices(int $clientId, array $input): array {
    $status = $input['status'] ?? '';
    $page = (int) ($input['page'] ?? 1);
    $limit = min((int) ($input['limit'] ?? 20), 50);

    $query = Capsule::table('tblhosting')
        ->where('userid', $clientId)
        ->select('id', 'domain', 'domainstatus', 'nextduedate', 'billingcycle');

    if ($status) {
        $query->where('domainstatus', $status);
    }

    $total = $query->count();
    $services = $query
        ->orderBy('id', 'DESC')
        ->offset(($page - 1) * $limit)
        ->limit($limit)
        ->get();

    return [
        'data' => $services,
        'meta' => [
            'page' => $page,
            'limit' => $limit,
            'total' => $total,
        ],
    ];
}

function getServiceDetails(int $clientId, int $serviceId): array {
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->where('userid', $clientId)
        ->first();

    if (!$service) {
        throw new \Exception('Service not found');
    }

    // Get product details
    $product = Capsule::table('tblproducts')->find($service->pid);

    // Get module data if applicable
    $moduleData = [];
    if ($service->username) {
        $moduleData = Capsule::table('mod_myservice_data')
            ->where('service_id', $serviceId)
            ->first() ?: [];
    }

    return [
        'service' => $service,
        'product' => $product,
        'module_data' => $moduleData,
    ];
}

function updateProfile(int $clientId, array $input): array {
    check_token('WHMCS.default'); // Client area token

    $allowedFields = ['firstname', 'lastname', 'companyname', 'phonenumber'];
    $updateData = array_intersect_key($input, array_flip($allowedFields));

    if (empty($updateData)) {
        throw new \Exception('No valid fields to update');
    }

    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update($updateData);

    logActivity("Client {$clientId} updated profile via API");

    return ['message' => 'Profile updated successfully'];
}

function changePassword(int $clientId, array $input): array {
    check_token('WHMCS.default');

    $currentPassword = $input['current_password'] ?? '';
    $newPassword = $input['new_password'] ?? '';

    if (empty($newPassword) || strlen($newPassword) < 8) {
        throw new \Exception('Password must be at least 8 characters');
    }

    // Verify current password
    $client = Capsule::table('tblclients')->find($clientId);
    if (!verifyClientPassword($clientId, $currentPassword)) {
        throw new \Exception('Current password is incorrect');
    }

    // Update password
    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update(['password' => password_hash($newPassword, PASSWORD_DEFAULT)]);

    logActivity("Client {$clientId} changed password");

    return ['message' => 'Password changed successfully'];
}
```

### 2. Custom Client Portal Pages

```php
<?php
// modules/addons/myaddon/client/index.php

if (!defined("WHMCS")) { die("Direct access denied"); }

// Client authentication check
$clientId = $_SESSION['uid'] ?? 0;
if (!$clientId) {
    header('Location: ' . \App::getSystemURL() . 'clientarea.php');
    exit;
}

// Get client data
$client = Capsule::table('tblclients')->find($clientId);

// Determine view
$view = $_GET['view'] ?? 'dashboard';

switch ($view) {
    case 'dashboard':
        include __DIR__ . '/views/dashboard.php';
        break;
    case 'services':
        include __DIR__ . '/views/services.php';
        break;
    case 'billing':
        include __DIR__ . '/views/billing.php';
        break;
    case 'settings':
        include __DIR__ . '/views/settings.php';
        break;
    case 'api':
        include __DIR__ . '/api.php';
        break;
    default:
        include __DIR__ . '/views/dashboard.php';
}
```

### 3. Client Portal Dashboard View

```php
<?php
// modules/addons/myaddon/client/views/dashboard.php

use WHMCS\Database\Capsule;

// Get statistics
$stats = [
    'active_services' => Capsule::table('tblhosting')
        ->where('userid', $clientId)
        ->where('domainstatus', 'Active')
        ->count(),
    'open_tickets' => Capsule::table('tbltickets')
        ->where('userid', $clientId)
        ->whereIn('status', ['Open', 'Answered'])
        ->count(),
    'pending_invoices' => Capsule::table('tblinvoices')
        ->where('userid', $clientId)
        ->where('status', 'Unpaid')
        ->count(),
];

// Get recent orders
$recentOrders = Capsule::table('tblorders')
    ->where('userid', $clientId)
    ->orderBy('date', 'DESC')
    ->limit(5)
    ->get();

// Get upcoming renewals
$upcomingRenewals = Capsule::table('tblhosting')
    ->where('userid', $clientId)
    ->where('nextduedate', '<=', date('Y-m-d', strtotime('+30 days')))
    ->where('nextduedate', '>=', date('Y-m-d'))
    ->where('domainstatus', 'Active')
    ->orderBy('nextduedate')
    ->limit(5)
    ->get();

// Assign to Smarty
$smarty = new \WHMCS\Smarty\Frontend();
$smarty->assign('client', $client);
$smarty->assign('stats', $stats);
$smarty->assign('recent_orders', $recentOrders);
$smarty->assign('upcoming_renewals', $upcomingRenewals);

echo $smarty->fetch(__DIR__ . '/../templates/dashboard.tpl');
```

### 4. Client Service Management

```php
<?php
// modules/addons/myaddon/client/views/services.php

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    check_token('WHMCS.default');

    $action = $_POST['action'] ?? '';

    switch ($action) {
        case 'cancel_service':
            handleServiceCancellation($clientId, (int) $_POST['service_id']);
            break;
        case 'upgrade_service':
            handleServiceUpgrade($clientId, (int) $_POST['service_id']);
            break;
        case 'request_transfer':
            handleTransferRequest($clientId, (int) $_POST['service_id']);
            break;
    }
}

// Get all services with details
$services = Capsule::table('tblhosting as h')
    ->join('tblproducts as p', 'h.pid', '=', 'p.id')
    ->leftJoin('tblpricing as pr', function($join) {
        $join->on('pr.type', '=', \DB::raw("'product'"))
             ->on('pr.currency', '=', 'h.billingcycle');
    })
    ->where('h.userid', $clientId)
    ->select(
        'h.*',
        'p.name as product_name',
        'p.gid as group_id',
        'pr.msetupfee',
        'pr.qsetupfee',
        'pr.ssetupfee',
        'pr.asetupfee',
        'pr.bsetupfee',
        'pr.tsetupfee'
    )
    ->orderBy('h.nextduedate', 'ASC')
    ->get();

// Group services by product group
$groupedServices = [];
foreach ($services as $service) {
    $groupId = $service->group_id ?: 0;
    if (!isset($groupedServices[$groupId])) {
        $groupedServices[$groupId] = [
            'group_name' => getGroupName($groupId),
            'services' => [],
        ];
    }
    $groupedServices[$groupId]['services'][] = $service;
}

$smarty->assign('grouped_services', $groupedServices);
echo $smarty->fetch(__DIR__ . '/../templates/services.tpl');

function handleServiceCancellation(int $clientId, int $serviceId): void {
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->where('userid', $clientId)
        ->first();

    if (!$service) {
        $_SESSION['flash_error'] = 'Service not found';
        return;
    }

    $reason = $_POST['cancel_reason'] ?? '';
    $immediate = isset($_POST['immediate']);

    // Create cancellation request
    Capsule::table('tblcancelrequests')->insert([
        'relid' => $serviceId,
        'type' => $immediate ? 'immediate' : 'endofbilling',
        'reason' => $reason,
        'requested' => date('Y-m-d H:i:s'),
    ]);

    logActivity("Client {$clientId} requested cancellation for service {$serviceId}");

    $_SESSION['flash_message'] = 'Cancellation request submitted';
}
```

### 5. Client Billing Page

```php
<?php
// modules/addons/myaddon/client/views/billing.php

use WHMCS\Database\Capsule;

// Get active balance
$balance = Capsule::table('tblclients')->find($clientId);
$accountBalance = $balance->credit;

// Get invoices
$status = $_GET['status'] ?? '';
$invoices = Capsule::table('tblinvoices')
    ->where('userid', $clientId)
    ->when($status, function($query) use ($status) {
        return $query->where('status', $status);
    })
    ->orderBy('id', 'DESC')
    ->limit(50)
    ->get();

// Calculate totals
$totals = [
    'paid' => Capsule::table('tblinvoices')
        ->where('userid', $clientId)
        ->where('status', 'Paid')
        ->sum('total'),
    'pending' => Capsule::table('tblinvoices')
        ->where('userid', $clientId)
        ->where('status', 'Unpaid')
        ->sum('total'),
];

// Get payment methods
$paymentMethods = Capsule::table('tblpaymentgateways')
    ->where('setting', 'name')
    ->where('gateway', '!=', '')
    ->groupBy('gateway')
    ->pluck('gateway')
    ->toArray();

$smarty->assign('account_balance', $accountBalance);
$smarty->assign('invoices', $invoices);
$smarty->assign('totals', $totals);
$smarty->assign('payment_methods', $paymentMethods);
$smarty->assign('status_filter', $status);

echo $smarty->fetch(__DIR__ . '/../templates/billing.tpl');
```

### 6. Client Settings Page

```php
<?php
// modules/addons/myaddon/client/views/settings.php

use WHMCS\Database\Capsule;

// Handle form submission
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    check_token('WHMCS.default');

    $section = $_POST['section'] ?? '';

    switch ($section) {
        case 'profile':
            updateClientProfile($clientId, $_POST);
            break;
        case 'password':
            updateClientPassword($clientId, $_POST);
            break;
        case 'contacts':
            manageContacts($clientId, $_POST);
            break;
        case 'payment':
            updatePaymentPreferences($clientId, $_POST);
            break;
        case 'notification':
            updateNotificationPreferences($clientId, $_POST);
            break;
    }
}

// Load current settings
$client = Capsule::table('tblclients')->find($clientId);
$contacts = Capsule::table('tblcontacts')
    ->where('userid', $clientId)
    ->get();
$notifications = getNotificationPreferences($clientId);

$smarty->assign('client', $client);
$smarty->assign('contacts', $contacts);
$smarty->assign('notification_prefs', $notifications);

echo $smarty->fetch(__DIR__ . '/../templates/settings.tpl');

function updateClientProfile(int $clientId, array $data): void {
    $allowed = ['firstname', 'lastname', 'companyname', 'phonenumber', 'address1', 'address2', 'city', 'state', 'postcode', 'country'];

    $update = array_intersect_key($data, array_flip($allowed));

    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update($update);

    $_SESSION['flash_message'] = 'Profile updated successfully';
    logActivity("Client {$clientId} updated profile");
}

function updateClientPassword(int $clientId, array $data): void {
    // Verify current password
    $client = Capsule::table('tblclients')->find($clientId);

    if (!password_verify($data['current_password'], $client->password)) {
        throw new \Exception('Current password is incorrect');
    }

    if ($data['new_password'] !== $data['confirm_password']) {
        throw new \Exception('Passwords do not match');
    }

    if (strlen($data['new_password']) < 8) {
        throw new \Exception('Password must be at least 8 characters');
    }

    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update(['password' => password_hash($data['new_password'], PASSWORD_DEFAULT)]);

    $_SESSION['flash_message'] = 'Password changed successfully';
    logActivity("Client {$clientId} changed password");
}
```

## Client Portal Templates

### 1. Main Layout Template

```html
<!-- templates/client/layout.tpl -->

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{$page_title|default:'Client Portal'}</title>
    <link rel="stylesheet" href="{$base_path}/templates/client/style.css">
</head>
<body>
    <header class="client-header">
        <div class="container">
            <div class="logo">
                <img src="{$company_logo}" alt="{$company_name}">
            </div>
            <nav class="client-nav">
                <a href="clientarea.php">Dashboard</a>
                <a href="?view=services">Services</a>
                <a href="?view=billing">Billing</a>
                <a href="?view=settings">Settings</a>
                <a href="logout.php">Logout</a>
            </nav>
        </div>
    </header>

    <main class="client-main">
        <div class="container">
            {if isset($flash_message)}
            <div class="alert alert-success">{$flash_message}</div>
            {/if}

            {if isset($flash_error)}
            <div class="alert alert-danger">{$flash_error}</div>
            {/if}

            {$content}
        </div>
    </main>

    <footer class="client-footer">
        <div class="container">
            <p>&copy; {year} {$company_name}. All rights reserved.</p>
        </div>
    </footer>

    <script src="{$base_path}/templates/client/app.js"></script>
</body>
</html>
```

### 2. Dashboard Template

```html
<!-- templates/client/dashboard.tpl -->

<div class="dashboard">
    <h1>Welcome back, {$client->firstname}!</h1>

    <!-- Stats Grid -->
    <div class="row stats-grid">
        <div class="col-md-3">
            <div class="stat-card">
                <i class="fa fa-server"></i>
                <h3>{$stats.active_services}</h3>
                <p>Active Services</p>
            </div>
        </div>
        <div class="col-md-3">
            <div class="stat-card">
                <i class="fa fa-ticket"></i>
                <h3>{$stats.open_tickets}</h3>
                <p>Open Tickets</p>
            </div>
        </div>
        <div class="col-md-3">
            <div class="stat-card">
                <i class="fa fa-file-text"></i>
                <h3>{$stats.pending_invoices}</h3>
                <p>Pending Invoices</p>
            </div>
        </div>
        <div class="col-md-3">
            <div class="stat-card">
                <i class="fa fa-credit-card"></i>
                <h3>{$client->credit|currency_format}</h3>
                <p>Account Balance</p>
            </div>
        </div>
    </div>

    <!-- Upcoming Renewals -->
    <div class="panel">
        <h3>Upcoming Renewals</h3>
        {if $upcoming_renewals}
        <table class="table">
            <thead>
                <tr>
                    <th>Service</th>
                    <th>Next Due</th>
                    <th>Amount</th>
                    <th>Action</th>
                </tr>
            </thead>
            <tbody>
                {foreach $upcoming_renewals as $renewal}
                <tr>
                    <td>{$renewal->domain}</td>
                    <td>{$renewal->nextduedate}</td>
                    <td>{$renewal->amount|currency_format}</td>
                    <td>
                        <a href="clientarea.php?action=productdetails&id={$renewal->id}"
                           class="btn btn-sm btn-default">View</a>
                    </td>
                </tr>
                {/foreach}
            </tbody>
        </table>
        {else}
        <p class="text-muted">No upcoming renewals in the next 30 days.</p>
        {/if}
    </div>
</div>
```

## Client Authentication

```php
<?php
// Custom client authentication hook

add_hook('ClientLogin', 1, function($vars) {
    $clientId = $vars['userid'];

    // Log login
    logActivity("Client {$clientId} logged in");

    // Update last login
    \WHMCS\Database\Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update(['lastlogin' => date('Y-m-d H:i:s')]);

    // Set session data
    $_SESSION['client_last_activity'] = time();
});

add_hook('ClientLogout', 1, function($vars) {
    if (isset($_SESSION['client_last_activity'])) {
        $duration = time() - $_SESSION['client_last_activity'];
        logActivity("Client session ended. Duration: {$duration} seconds");
    }
});
```

## Checklist

- [ ] Client area pages check authentication
- [ ] CSRF tokens on all forms
- [ ] Input validation on all user inputs
- [ ] Proper error handling with user-friendly messages
- [ ] Responsive design for mobile devices
- [ ] Activity logging for client actions
- [ ] API rate limiting
- [ ] Security headers on API responses

---

**Related Skills:**
- whmcs-clientarea-builder
- whmcs-ajax-patterns
- whmcs-api-integration
- whmcs-security-hardening
- whmcs-email-templates

**Reference:**
- WHMCS Client Area: https://developers.whmcs.com/client-area/