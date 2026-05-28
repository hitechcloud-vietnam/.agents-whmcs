# WHMCS Admin Area Customization

**Version:** 8.x
**Updated:** 2026-05-28
**Related Skills:** `client-area-theming`, `hooks-reference`, `smarty-template-reference`

## Overview

Customizing the WHMCS admin area involves modifying templates, adding custom CSS/JS, creating admin hooks, and extending functionality through the admin directory structure.

## Directory Structure

```
whmcs/
├── admin/
│   ├── templates/
│   │   ├── default/
│   │   │   ├── header.tpl
│   │   │   ├── footer.tpl
│   │   │   ├── sidebar.tpl
│   │   │   └── login.tpl
│   │   └── your-theme/
│   ├── css/
│   ├── js/
│   ├── images/
│   └── includes/
│       └── functions.php
├── resources/
│   └── admin/
```

## Template Customization

### Header Template

```smarty
{* admin/templates/default/header.tpl *}
<!DOCTYPE html>
<html>
<head>
    <meta charset="{$charset}">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{$page_title}</title>

    {* Include default styles *}
    <link rel="stylesheet" href="{$BASE_PATH_Admin}/css/bootstrap.min.css">
    <link rel="stylesheet" href="{$BASE_PATH_Admin}/css/font-awesome.min.css">

    {* Custom admin CSS *}
    <link rel="stylesheet" href="{$BASE_PATH_Admin}/css/admin-custom.css">

    {* Admin-specific JavaScript *}
    <script src="{$BASE_PATH_Admin}/js/admin-common.js"></script>

    {$headoutput}
</head>
<body class="{$body_class}">
```

### Footer Template

```smarty
{* admin/templates/default/footer.tpl *}
    {$footoutput}

    {* Default scripts *}
    <script src="{$BASE_PATH_Admin}/js/jquery.min.js"></script>
    <script src="{$BASE_PATH_Admin}/js/bootstrap.min.js"></script>
    <script src="{$BASE_PATH_Admin}/js/admin-custom.js"></script>

    {* WHMCS utility functions *}
    <script>
        WHMCS.adminutils.init();
        WHMCS.dataTable.init();
    </script>
</body>
</html>
```

### Sidebar Navigation

```smarty
{* admin/templates/default/sidebar.tpl *}
<nav class="navbar-default navbar-static-side" role="navigation">
    <div class="sidebar-collapse">
        <ul class="nav" id="side-menu">
            <li class="nav-header">
                <div class="dropdown profile-element">
                    <img src="{$company_logo_url}" alt="Logo" class="admin-logo">
                    <span class="block m-t-xs">
                        <strong>{$admin_full_name}</strong>
                    </span>
                </div>
            </li>

            {* System Status *}
            <li>
                <a href="{$BASE_PATH_Admin}/systemhealth.php">
                    <i class="fa fa-dashboard"></i>
                    <span>System Health</span>
                    <span class="label label-success pull-right">OK</span>
                </a>
            </li>

            {* Clients *}
            <li class="dropdown">
                <a href="#" class="dropdown-toggle" data-toggle="dropdown">
                    <i class="fa fa-users"></i>
                    <span>Clients</span>
                    <span class="caret"></span>
                </a>
                <ul class="dropdown-menu">
                    <li><a href="{$BASE_PATH_Admin}/clients.php">View All</a></li>
                    <li><a href="{$BASE_PATH_Admin}/clientsadd.php">Add New</a></li>
                    <li><a href="{$BASE_PATH_Admin}/clientsummary.php">Summary</a></li>
                </ul>
            </li>
        </ul>
    </div>
</nav>
```

## Custom Admin CSS

```css
/* admin/css/admin-custom.css */

/* Custom brand colors */
:root {
    --admin-primary: #1a73e8;
    --admin-secondary: #5f6368;
    --admin-success: #34a853;
    --admin-danger: #ea4335;
    --admin-warning: #fbbc04;
}

/* Sidebar customization */
.sidebar {
    background: linear-gradient(180deg, #1a1a2e 0%, #16213e 100%);
}

.sidebar-menu > li > a {
    padding: 12px 20px;
    transition: all 0.3s ease;
}

.sidebar-menu > li > a:hover {
    background: rgba(255, 255, 255, 0.1);
}

/* Card customization */
.admin-card {
    border: none;
    border-radius: 12px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    transition: transform 0.2s;
}

.admin-card:hover {
    transform: translateY(-2px);
}

/* Status badges */
.status-badge {
    padding: 4px 12px;
    border-radius: 20px;
    font-size: 12px;
    font-weight: 500;
}

.status-badge.active {
    background: #e6f4ea;
    color: #1e8e3e;
}

/* Custom table styles */
.admin-table {
    border-collapse: separate;
    border-spacing: 0;
}

.admin-table thead th {
    background: var(--admin-primary);
    color: white;
    border: none;
    padding: 12px;
}

.admin-table tbody tr:hover {
    background: #f8f9fa;
}

/* Quick action buttons */
.quick-action {
    width: 60px;
    height: 60px;
    border-radius: 12px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 24px;
}
```

## Custom Admin JavaScript

```javascript
// admin/js/admin-custom.js

(function() {
    'use strict';

    // Initialize on document ready
    $(document).ready(function() {
        initCustomFeatures();
        initDataTablesExtensions();
        initSidebarToggles();
    });

    function initCustomFeatures() {
        // Add custom tooltips
        $('[data-toggle="custom-tooltip"]').tooltip({
            placement: 'top',
            container: 'body'
        });

        // Confirm destructive actions
        $('[data-confirm-action]').on('click', function(e) {
            const message = $(this).data('confirm-message') || 'Are you sure?';
            if (!confirm(message)) {
                e.preventDefault();
            }
        });
    }

    function initDataTablesExtensions() {
        // Custom column visibility
        if ($.fn.DataTable) {
            $.extend(true, $.fn.DataTable.defaults, {
                'dom': 'Bfrtip',
                'buttons': ['copy', 'csv', 'excel', 'pdf', 'print'],
                'pageLength': 25,
                'order': [[0, 'desc']]
            });
        }
    }

    function initSidebarToggles() {
        // Collapsible sidebar sections
        $('.sidebar-section-toggle').on('click', function(e) {
            e.preventDefault();
            const target = $(this).data('target');
            $(target).toggleClass('collapsed');
        });
    }

    // Custom client search
    window.adminClientSearch = function(query) {
        return $.ajax({
            url: 'clients.php',
            data: {
                'token': csrfToken,
                'search': query,
                'ajax': 1
            },
            dataType: 'json'
        });
    };

})();
```

## Admin Hooks

### AdminAreaHeader

```php
// includes/hooks/admin_customization.php

use WHMCS\View\Markup\AdminOrderStatusIcon;

add_hook('AdminAreaHeader', 1, function($vars) {
    return <<<HTML
    <style>
        /* Custom admin header styles */
        .admin-header-badge {
            background: #1a73e8;
            color: white;
            padding: 2px 8px;
            border-radius: 10px;
            font-size: 11px;
        }
    </style>
    <script>
        // Custom header scripts
        console.log('Admin area loaded');
    </script>
    HTML;
});
```

### AdminAreaClientSummaryPage

```php
add_hook('AdminAreaClientSummaryPage', 1, function($vars) {
    $clientId = $vars['userid'];
    $clientModel = $vars['client'];

    // Get custom stats
    $stats = [
        'total_orders' => $clientModel->orders()->count(),
        'active_services' => $clientModel->services()->where('domainstatus', 'Active')->count(),
        'total_spent' => $clientModel->invoices()->where('status', 'Paid')->sum('total'),
    ];

    return <<<HTML
    <div class="panel panel-default">
        <div class="panel-heading">
            <h3 class="panel-title">Custom Statistics</h3>
        </div>
        <div class="panel-body">
            <div class="row">
                <div class="col-sm-4">
                    <div class="stat-value">{$stats['total_orders']}</div>
                    <div class="stat-label">Total Orders</div>
                </div>
                <div class="col-sm-4">
                    <div class="stat-value">{$stats['active_services']}</div>
                    <div class="stat-label">Active Services</div>
                </div>
                <div class="col-sm-4">
                    <div class="stat-value">\${$stats['total_spent']}</div>
                    <div class="stat-label">Total Spent</div>
                </div>
            </div>
        </div>
    </div>
    HTML;
});
```

### AdminAreaFooter

```php
add_hook('AdminAreaFooter', 1, function($vars) {
    return <<<HTML
    <script>
        // Add custom footer functionality
        $(document).ajaxComplete(function(event, xhr, settings) {
            if (settings.url.includes('whmcs')) {
                console.log('WHMCS API call completed');
            }
        });
    </script>
    HTML;
});
```

### AdminClientEdit

```php
add_hook('AdminClientEdit', 1, function($vars) {
    $clientId = $vars['userid'];

    // Validate custom fields
    $customFieldValue = $_POST['customfield']['your_field_id'] ?? '';

    if (empty($customFieldValue)) {
        return [
            'error' => 'Custom field is required',
            'field' => 'customfield_your_field_id'
        ];
    }

    return true;
});
```

## Custom Admin Pages

### Creating Custom Admin Page

```php
// admin/custom/example_page.php

define('ADMINAREA', true);
require('../../init.php');

$title = 'Custom Admin Page';
$templatefile = 'example_page';

$sidebar = 'custom';
$breadcrumb = [
    ['name' => 'Home', 'url' => 'index.php'],
    ['name' => 'Custom', 'url' => 'custom.php'],
    ['name' => 'Example', 'url' => ''],
];

// Get data
$stats = [
    'total_clients' => Capsule::table('tblclients')->count(),
    'active_services' => Capsule::table('tblhosting')->where('domainstatus', 'Active')->count(),
    'monthly_revenue' => Capsule::table('tblinvoices')
        ->where('status', 'Paid')
        ->whereMonth('date', date('m'))
        ->sum('total'),
];

// Assign to template
$smartyvalues = [
    'stats' => $stats,
    'recent_activity' => getRecentActivity(),
];

// Output
$template = new WHMCS\Admin\ApplicationSupport\View\Html\Template($templatefile);
$template->setPageTitle($title);
$template->setSidebar($sidebar);
$template->setBreadcrumbs($breadcrumb);
$template->setTemplateVariables($templatevalues);
$template->setFaq($faq);
$template->render();
```

## Admin Dashboard Widgets

```php
// includes/hooks/admin_dashboard_widgets.php

add_hook('AdminHomepage', 1, function($vars) {
    return [
        'displayFunction' => 'renderCustomWidget',
        'strings' => [
            'title' => 'Custom Statistics',
        ],
        'requires' => [],
    ];
});

function renderCustomWidget() {
    $data = Capsule::table('tblorders')
        ->select('status', Capsule::raw('COUNT(*) as count'))
        ->groupBy('status')
        ->get();

    echo '<div class="admin-widget">';
    echo '<h4>Order Status</h4>';
    echo '<ul class="list-unstyled">';
    foreach ($data as $row) {
        $color = match($row->status) {
            'Active' => 'success',
            'Pending' => 'warning',
            'Cancelled' => 'danger',
            default => 'default'
        };
        echo "<li><span class='label label-{$color}'>{$row->status}</span> {$row->count}</li>";
    }
    echo '</ul>';
    echo '</div>';
}
```

## Customizing Admin Login

```php
// includes/hooks/admin_login_customization.php

// Custom login logo
add_hook('AdminLoginLogo', 1, function($vars) {
    return '<img src="https://your-cdn.com/custom-admin-logo.png" alt="Admin Login" class="login-logo">';
});

// Custom login footer
add_hook('AdminLoginFooter', 1, function($vars) {
    return '<p class="text-muted">Custom login footer text</p>';
});
```

## Best Practices

1. **Use Child Themes**: Create a custom admin theme instead of modifying default
2. **Hooks Over Core Mods**: Prefer hooks for functionality changes
3. **CSS Specificity**: Use specific selectors to avoid conflicts
4. **CSRF Protection**: Always use csrfToken in AJAX calls
5. **Template Variables**: Check variables exist before using them
6. **Version Compatibility**: Test admin changes across WHMCS versions

## Related Documentation

- [Client Area Theming](client-area-theming.md)
- [Hooks Reference](hooks-reference.md)
- [Smarty Template Reference](smarty-template-reference.md)
