# WHMCS Client Portal

## Overview
Master skill for customizing the WHMCS client portal. Covers dashboard customization, client area pages, and user experience improvements.

## Client Portal Hooks

```php
<?php
// /includes/hooks/client_portal_hooks.php

add_hook("ClientAreaPage", 1, function(array $params) {
    if ($params["templatefile"] === "clienthome") {
        add_hook_content("homepage_widgets", function() {
            return renderCustomDashboardWidget();
        });
    }
    return $params;
});

add_hook("ClientAreaHeaderOutput", 1, function(array $params) {
    $params["customJS"] = '<script src="custom-portal.js"></script>';
    return $params;
});

function renderCustomDashboardWidget(): string
{
    $html = '<div class="panel panel-default">';
    $html .= '<div class="panel-heading">Quick Stats</div>';
    $html .= '<div class="panel-body">';
    $html .= '<p>Active Services: ' . getActiveServiceCount() . '</p>';
    $html .= '<p>Open Tickets: ' . getOpenTicketCount() . '</p>';
    $html .= '</div></div>';
    return $html;
}
```

## Portal Template Customization

```php
<?php
// /includes/helpers/portal_helper.php

function customizeClientDashboard(int $clientId): array
{
    $client = \WHMCS\User\Client::find($clientId);
    
    return [
        "greeting" => getGreeting($client),
        "stats" => getClientStats($clientId),
        "recent_activity" => getRecentActivity($clientId),
        "notifications" => getClientNotifications($clientId),
        "quick_actions" => getQuickActions($client),
    ];
}

function getGreeting(\WHMCS\User\Client $client): string
{
    $hour = (int)date("H");
    
    if ($hour < 12) {
        return "Good morning, {$client->firstname}";
    } elseif ($hour < 18) {
        return "Good afternoon, {$client->firstname}";
    } else {
        return "Good evening, {$client->firstname}";
    }
}

function getClientStats(int $clientId): array
{
    return [
        "active_services" => \Illuminate\Database\Capsule\Manager::table("tblhosting")
            ->where("userid", $clientId)
            ->where("domainstatus", "Active")
            ->count(),
        "pending_invoices" => \Illuminate\Database\Capsule\Manager::table("tblinvoices")
            ->where("userid", $clientId)
            ->where("status", "Unpaid")
            ->count(),
        "open_tickets" => \Illuminate\Database\Capsule\Manager::table("tbltickets")
            ->where("userid", $clientId)
            ->whereIn("status", ["Open", "Answered"])
            ->count(),
        "total_spent" => \Illuminate\Database\Capsule\Manager::table("tblinvoices")
            ->where("userid", $clientId)
            ->where("status", "Paid")
            ->sum("total"),
    ];
}
```

## Best Practices

1. **Responsive Design**: Ensure mobile compatibility
2. **Performance**: Optimize page load times
3. **Customization**: Allow user preferences
4. **Navigation**: Clear and intuitive menus
5. **Quick Actions**: Provide shortcuts for common tasks
6. **Notifications**: Show important alerts
7. **Security**: Maintain security standards
8. **Accessibility**: Follow WCAG guidelines
