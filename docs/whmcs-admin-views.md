# WHMCS Admin Views

## Overview

Customize the WHMCS admin area with custom views, dashboards, and pages.

## Custom Admin Pages

### Basic Admin Page Structure

```php
<?php
use WHMCS\View\Markup\MarkupRenderer;
use WHMCS\Table\Table;

function yourmodule_output(array $vars): void
{
    // Check permission
    if (!checkPermission('Your Permission', true)) {
        echo '<div class="alert alert-danger">Access denied</div>';
        return;
    }
    
    // Handle form submission
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
        processForm($_POST);
        redir('module=yourmodule&success=1');
    }
    
    // Get data
    $data = getModuleData();
    
    // Render template
    echo '<div class="module-header">';
    echo '<h1>Your Module</h1>';
    echo '</div>';
    
    if (isset($_GET['success'])) {
        echo '<div class="alert alert-success">Action completed successfully</div>';
    }
    
    // Output content
    renderContent($data);
}
```

### Admin View with DataTable

```php
<?php
function yourmodule_output(array $vars): void
{
    $table = new Table();
    
    $table->addColumn('ID', 'id');
    $table->addColumn('Client', 'clientName');
    $table->addColumn('Service', 'serviceName');
    $table->addColumn('Status', 'status');
    $table->addColumn('Actions', 'actions');
    
    // Get data with pagination
    $page = (int) ($_GET['page'] ?? 1);
    $data = getModuleData($page);
    
    foreach ($data['items'] as $item) {
        $table->addRow([
            'id' => $item->id,
            'clientName' => '<a href="clientssummary.php?userid=' . $item->userid . '">' . 
                           htmlspecialchars($item->clientName) . '</a>',
            'serviceName' => htmlspecialchars($item->serviceName),
            'status' => '<span class="label label-' . strtolower($item->status) . '">' . 
                       htmlspecialchars($item->status) . '</span>',
            'actions' => '<a href="?module=yourmodule&action=edit&id=' . $item->id . '" class="btn btn-xs btn-default">Edit</a>'
        ]);
    }
    
    echo '<div class="content-header">';
    echo '<div class="pull-right">';
    echo '<a href="?module=yourmodule&action=add" class="btn btn-primary">Add New</a>';
    echo '</div>';
    echo '<h2>Module Items</h2>';
    echo '</div>';
    
    echo $table->output();
}
```

### Admin Page with Charts

```php
<?php
function dashboard_output(): void
{
    echo '<div class="row">';
    
    // Stats boxes
    $stats = getDashboardStats();
    
    foreach ($stats as $stat) {
        echo '<div class="col-sm-6 col-md-3">';
        echo '<div class="panel panel-default">';
        echo '<div class="panel-body text-center">';
        echo '<h3>' . number_format($stat['value']) . '</h3>';
        echo '<p>' . $stat['label'] . '</p>';
        echo '</div>';
        echo '</div>';
        echo '</div>';
    }
    
    echo '</div>';
    
    // Chart
    echo '<div class="row">';
    echo '<div class="col-md-8">';
    echo '<div class="panel panel-default">';
    echo '<div class="panel-heading">Revenue Chart</div>';
    echo '<div class="panel-body">';
    echo '<canvas id="revenueChart"></canvas>';
    echo '</div>';
    echo '</div>';
    echo '</div>';
    
    echo '<div class="col-md-4">';
    echo '<div class="panel panel-default">';
    echo '<div class="panel-heading">Top Clients</div>';
    echo '<div class="panel-body">';
    echo renderTopClients();
    echo '</div>';
    echo '</div>';
    echo '</div>';
    echo '</div>';
}
```

### Admin Page Hooks

```php
<?php
// Add sidebar widget to admin dashboard
add_hook('AdminHomepage', 1, function($vars) {
    $recentItems = getRecentModuleItems();
    
    return [
        'sidebar' => [
            [
                'title' => 'Recent Items',
                'template' => 'widgets/recent-items',
                'data' => ['items' => $recentItems],
            ],
        ],
    ];
});

// Add tab to client profile
add_hook('ClientDetailsTabs', 1, function($vars) {
    return [
        'tabName' => 'Your Module',
        'tabId' => 'module-tab',
        'label' => 'Your Module',
        'order' => 100,
    ];
});

add_hook('ClientDetailsTabsContent', 1, function($vars) {
    if ($vars['tab'] !== 'module-tab') {
        return;
    }
    
    $clientId = $vars['userid'];
    $items = getClientItems($clientId);
    
    return [
        'template' => 'client/module-items',
        'vars' => ['items' => $items],
    ];
});
```

### Admin Navigation

```php
<?php
// Add to admin navigation
add_hook('AdminAreaNav', 1, function($vars) {
    $adminId = (int) $_SESSION['adminid'];
    $roleId = (int) $_SESSION['admin_role'];
    
    // Only show for certain roles
    if ($roleId !== 1) {
        return ['add' => []];
    }
    
    return [
        'add' => [
            [
                'name' => 'module',
                'label' => 'Your Module',
                'uri' => 'addonmodules.php?module=yourmodule',
                'icon' => 'fa fa-cog',
                'order' => 50,
            ],
        ],
    ];
});
```

### Bulk Actions

```php
<?php
function renderBulkActions(): void
{
    echo '<form method="post" action="?module=yourmodule&action=bulk">';
    echo '<input type="hidden" name="token" value="' . generate_token() . '">';
    
    echo '<div class="row">';
    echo '<div class="col-md-12">';
    
    // Bulk action selector
    echo '<div class="pull-left" style="margin-bottom: 10px;">';
    echo '<select name="bulk_action" class="form-control" style="width: 200px; display: inline;">';
    echo '<option value="">Select Action</option>';
    echo '<option value="delete">Delete Selected</option>';
    echo '<option value="export">Export Selected</option>';
    echo '<option value="update">Update Status</option>';
    echo '</select>';
    echo ' <button type="submit" class="btn btn-primary">Apply</button>';
    echo '</div>';
    
    // Select all checkbox
    echo '<div class="pull-right">';
    echo '<input type="checkbox" id="select-all"> Select All';
    echo '</div>';
    
    echo '</div>';
    echo '</div>';
    
    // Table with checkboxes
    echo '<table class="table">';
    echo '<!-- table content with checkboxes -->';
    echo '</table>';
    
    echo '</form>';
}

if ($_SERVER['REQUEST_METHOD'] === 'POST' && $_GET['action'] === 'bulk') {
    check_token('WHMCS.admin.default');
    
    $selectedIds = $_POST['selected'] ?? [];
    $action = $_POST['bulk_action'] ?? '';
    
    if (empty($selectedIds)) {
        redir('module=yourmodule&error=no_selection');
    }
    
    switch ($action) {
        case 'delete':
            deleteItems($selectedIds);
            break;
        case 'export':
            exportItems($selectedIds);
            break;
        case 'update':
            updateItemsStatus($selectedIds);
            break;
    }
    
    redir('module=yourmodule&success=bulk_complete');
}
```

## Best Practices

1. **Check permissions** - Verify admin access
2. **Use CSRF tokens** - Protect forms
3. **Support pagination** - Don't load all data at once
4. **Use existing styles** - Match WHMCS design
5. **Include breadcrumbs** - Help navigation

## Related Documentation

- [WHMCS Widget Development](/docs/whmcs-widget-dev.md)
- [WHMCS JavaScript & AJAX](/docs/whmcs-javascript-ajax.md)