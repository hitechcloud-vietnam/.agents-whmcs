# WHMCS Admin Dashboard

## Overview
Guide for customizing the WHMCS admin dashboard. Covers widgets, statistics, and custom dashboard components.

## Dashboard Customization

### Add Dashboard Widget

```php
<?php
// /includes/hooks/admin_dashboard.php

add_hook("AdminHomeWidgets", 1, function(array $params) {
    $widgets = [];
    
    // Recent Orders Widget
    $widgets[] = [
        "title" => "Recent Orders",
        "template" => "widget_recent_orders.tpl",
        "columns" => "one-third",
        "data" => [
            "orders" => getRecentOrders(10)
        ]
    ];
    
    // Pending Tickets Widget
    $widgets[] = [
        "title" => "Pending Tickets",
        "template" => "widget_pending_tickets.tpl",
        "columns" => "one-third",
        "data" => [
            "tickets" => getPendingTickets(),
            "count" => countPendingTickets()
        ]
    ];
    
    // System Status Widget
    $widgets[] = [
        "title" => "System Status",
        "template" => "widget_system_status.tpl",
        "columns" => "one-third",
        "data" => [
            "checks" => performSystemChecks()
        ]
    ];
    
    return ["widgets" => $widgets];
});

function getRecentOrders(int $limit): array
{
    return Capsule::table("tblorders")
        ->join("tblclients", "tblorders.userid", "=", "tblclients.id")
        ->join("tblproducts", "tblorders.products", "=", "tblproducts.id")
        ->select(
            "tblorders.id",
            "tblorders.date",
            "tblorders.status",
            "tblclients.firstname",
            "tblclients.lastname",
            "tblclients.email",
            "tblproducts.name as product_name",
            "tblorders.total"
        )
        ->orderBy("tblorders.date", "desc")
        ->limit($limit)
        ->get();
}

function countPendingTickets(): int
{
    return Capsule::table("tbltickets")
        ->whereIn("status", ["Open", "Customer Reply"])
        ->count();
}

function performSystemChecks(): array
{
    $checks = [];
    
    // Database connection
    $checks[] = [
        "name" => "Database",
        "status" => checkDatabaseConnection() ? "ok" : "error",
        "message" => checkDatabaseConnection() ? "Connected" : "Connection failed"
    ];
    
    // Disk space
    $freeSpace = disk_free_space("/");
    $totalSpace = disk_total_space("/");
    $usedPercent = (($totalSpace - $freeSpace) / $totalSpace) * 100;
    
    $checks[] = [
        "name" => "Disk Space",
        "status" => $usedPercent > 90 ? "error" : ($usedPercent > 75 ? "warning" : "ok"),
        "message" => sprintf("%.1f%% used", $usedPercent)
    ];
    
    // Cron status
    $lastCron = Capsule::table("tblactivitylog")
        ->where("description", "like", "%Cron Job%")
        ->orderBy("id", "desc")
        ->value("date");
    
    $cronHealthy = $lastCron && 
                   (time() - strtotime($lastCron)) < 3600;
    
    $checks[] = [
        "name" => "Cron Job",
        "status" => $cronHealthy ? "ok" : "warning",
        "message" => $lastCron ? "Last run: " . $lastCron : "Never run"
    ];
    
    return $checks;
}
```

### Dashboard Widget Template

```smarty
<!-- /admin/templates/widget_recent_orders.tpl -->
<div class="admin-widget" id="widget-recent-orders">
    <div class="widget-header">
        <h3>{$title}</h3>
        <a href="orders.php" class="widget-link">View All</a>
    </div>
    <div class="widget-content">
        <table class="widget-table">
            <thead>
                <tr>
                    <th>Order</th>
                    <th>Client</th>
                    <th>Product</th>
                    <th>Amount</th>
                    <th>Status</th>
                </tr>
            </thead>
            <tbody>
                {foreach $data.orders as $order}
                    <tr>
                        <td>
                            <a href="orders.php?action=view&id={$order->id}">
                                #{$order->id}
                            </a>
                        </td>
                        <td>
                            <a href="clientssummary.php?userid={$order->userid}">
                                {$order->firstname} {$order->lastname}
                            </a>
                        </td>
                        <td>{$order->product_name}</td>
                        <td>{$order->total|formatCurrency}</td>
                        <td>
                            <span class="label label-{strtolower($order->status)}">
                                {$order->status}
                            </span>
                        </td>
                    </tr>
                {/foreach}
            </tbody>
        </table>
    </div>
</div>
```

### Custom Statistics

```php
add_hook("AdminHomeStats", 1, function(array $params) {
    return [
        "stats" => [
            [
                "label" => "Monthly Revenue",
                "value" => formatCurrency(getMonthlyRevenue()),
                "change" => getRevenueChange(),
                "icon" => "fa-dollar"
            ],
            [
                "label" => "New Clients (This Month)",
                "value" => getNewClientsThisMonth(),
                "change" => getNewClientsChange(),
                "icon" => "fa-users"
            ],
            [
                "label" => "Active Services",
                "value" => getActiveServicesCount(),
                "change" => null,
                "icon" => "fa-server"
            ],
            [
                "label" => "Open Tickets",
                "value" => getOpenTicketsCount(),
                "change" => null,
                "icon" => "fa-ticket"
            ]
        ]
    ];
});

function getMonthlyRevenue(): float
{
    $startOfMonth = date("Y-m-01 00:00:00");
    
    return Capsule::table("tblaccounts")
        ->where("date", ">=", $startOfMonth)
        ->sum("amountin") ?? 0;
}

function getRevenueChange(): ?float
{
    $currentMonth = getMonthlyRevenue();
    $lastMonth = getRevenueForMonth(date("Y-m", strtotime("-1 month")));
    
    if ($lastMonth == 0) return null;
    
    return (($currentMonth - $lastMonth) / $lastMonth) * 100;
}
```

### Quick Actions Widget

```php
add_hook("AdminHomeQuickActions", 1, function(array $params) {
    return [
        "actions" => [
            [
                "label" => "Add New Client",
                "href" => "clientsadd.php",
                "icon" => "fa-user-plus"
            ],
            [
                "label" => "Create Invoice",
                "href" => "invoices.php?action=create",
                "icon" => "fa-file-invoice"
            ],
            [
                "label" => "View Support Tickets",
                "href" => "supporttickets.php",
                "icon" => "fa-life-ring"
            ],
            [
                "label" => "System Health",
                "href" => "systemhealth.php",
                "icon" => "fa-heartbeat"
            ]
        ]
    ];
});
```

## Dashboard Hooks

```php
// Customize dashboard columns
add_hook("AdminHomeColumns", 1, function(array $params) {
    return ["columns" => 3]; // 1, 2, or 3 columns
});

// Add JS/CSS to dashboard
add_hook("AdminHomeResources", 1, function(array $params) {
    return [
        "javascript" => ["/assets/js/admin-dashboard.js"],
        "css" => ["/assets/css/admin-dashboard.css"]
    ];
});
```

## Best Practices

1. **Performance**: Minimize database queries in widgets
2. **Caching**: Cache widget data with short TTL
3. **User Preferences**: Allow users to customize widget layout
4. **Responsive**: Ensure widgets work on all screen sizes
5. **Consistency**: Match WHMCS admin styling
6. **Helpful Links**: Include links to detailed views
7. **Status Indicators**: Use clear status colors
8. **Refresh**: Support AJAX refresh for live data
