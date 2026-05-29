# WHMCS Admin Navigation

## Overview
Guide for customizing WHMCS admin area navigation. Covers menu items, submenus, and dynamic navigation based on permissions.

## Navigation Customization

### Add Menu Item

```php
<?php
// /includes/hooks/admin_navigation.php

add_hook("AdminNavbar", 1, function(array $params) {
    // Add main menu item
    $params["navbar"]->addItem(
        "Custom Menu",
        "fa-cog",
        "custom.php",
        [],
        "custom" // Badge/counter
    );
    
    return $params;
});

add_hook("AdminSecondaryNavbar", 1, function(array $params) {
    // Add dropdown to top right navbar
    $params["secondaryNavbar"]->addItem(
        "Quick Actions",
        [
            ["label" => "New Order", "href" => "orders.php?action=create"],
            ["label" => "New Client", "href" => "clientsadd.php"],
            ["label" => "New Ticket", "href" => "supporttickets.php?action=open"],
        ]
    );
    
    return $params;
});
```

### Add Submenu Items

```php
add_hook("AdminClientsMenu", 1, function(array $params) {
    $params["menu"]->addChild(
        "Import Clients",
        [
            "label" => "Import Clients",
            "href" => "clientsimport.php",
            "icon" => "fa-upload"
        ]
    );
    
    $params["menu"]->addChild(
        "Export Clients",
        [
            "label" => "Export Clients",
            "href" => "clientsexport.php",
            "icon" => "fa-download"
        ]
    );
    
    return $params;
});

add_hook("AdminOrdersMenu", 1, function(array $params) {
    $params["menu"]->addChild(
        "Order Settings",
        [
            "label" => "Settings",
            "href" => "settings.php?group=orders",
            "icon" => "fa-cog"
        ]
    );
    
    return $params;
});
```

### Dynamic Menu Based on Permissions

```php
add_hook("AdminNavbar", 1, function(array $params) {
    global $adminid;
    
    $admin = Capsule::table("tbladmins")->where("id", $adminid)->first();
    
    // Add reports menu only for admins with permission
    if (hasAdminPermission($adminid, "reports")) {
        $params["navbar"]->addItem(
            "Reports",
            "fa-chart-bar",
            "reports.php",
            [
                ["label" => "Sales Report", "href" => "reports.php?type=sales"],
                ["label" => "Activity Report", "href" => "reports.php?type=activity"],
                ["label" => "Custom Report", "href" => "reports.php?type=custom"]
            ]
        );
    }
    
    // Add system menu for super admins
    if ($admin->roleid == 1) {
        $params["navbar"]->addItem(
            "System",
            "fa-cogs",
            "system.php",
            [
                ["label" => "Diagnostics", "href" => "diagnostics.php"],
                ["label" => "Logs", "href" => "systemlogs.php"],
                ["label" => "API Access", "href" => "apiaccess.php"]
            ]
        );
    }
    
    return $params;
});

function hasAdminPermission(int $adminId, string $permission): bool
{
    $admin = Capsule::table("tbladmins")->where("id", $adminId)->first();
    
    $permissions = Capsule::table("tbladminperms")
        ->where("roleid", $admin->roleid)
        ->pluck("permid")
        ->toArray();
    
    $permId = Capsule::table("tblpermissions")
        ->where("name", $permission)
        ->value("id");
    
    return in_array($permId, $permissions);
}
```

### Badge/Counter Updates

```php
add_hook("AdminNavbarBadges", 1, function(array $params) {
    // Update badge counts
    $pendingTickets = Capsule::table("tbltickets")
        ->where("status", "Open")
        ->count();
    
    $pendingOrders = Capsule::table("tblorders")
        ->where("status", "Pending")
        ->count();
    
    return [
        "badges" => [
            "supporttickets" => $pendingTickets,
            "orders" => $pendingOrders
        ]
    ];
});
```

### Custom Menu Template

```smarty
<!-- /admin/templates/custom_nav_menu.tpl -->
{foreach $navItems as $item}
    <li class="{if $item.active}active{/if}">
        <a href="{$item.href}">
            <i class="fa {$item.icon}"></i>
            <span>{$item.label}</span>
            {if $item.badge}
                <span class="badge">{$item.badge}</span>
            {/if}
        </a>
        {if $item.children}
            <ul class="sub-menu">
                {foreach $item.children as $child}
                    <li>
                        <a href="{$child.href}">
                            {if $child.icon}
                                <i class="fa {$child.icon}"></i>
                            {/if}
                            {$child.label}
                        </a>
                    </li>
                {/foreach}
            </ul>
        {/if}
    </li>
{/foreach}
```

## Navigation Helpers

```php
// Get current page breadcrumbs
add_hook("AdminBreadcrumbs", 1, function(array $params) {
    $breadcrumbs = [
        ["label" => "Home", "href" => "index.php"],
    ];
    
    // Add page-specific breadcrumbs
    if (isset($_GET["action"]) && $_GET["action"] === "view") {
        $breadcrumbs[] = ["label" => "Client Details", "href" => "clientssummary.php"];
        $breadcrumbs[] = ["label" => "View", "active" => true];
    }
    
    return ["breadcrumbs" => $breadcrumbs];
});
```

## Best Practices

1. **Consistency**: Match WHMCS admin styling
2. **Permissions**: Hide items based on admin permissions
3. **Performance**: Minimize database queries in menu
4. **Badges**: Show meaningful counters
5. **Organize**: Group related items together
6. **Icons**: Use appropriate FontAwesome icons
7. **Accessibility**: Ensure keyboard navigation
8. **Responsive**: Work on mobile admin
