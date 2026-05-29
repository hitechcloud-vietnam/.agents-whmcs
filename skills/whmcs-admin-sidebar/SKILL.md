# WHMCS Admin Sidebar

## Overview
Guide for customizing WHMCS admin sidebar widgets. Covers sidebar panels, widgets, and collapsible sections.

## Sidebar Customization

### Add Sidebar Widget

```php
<?php
// /includes/hooks/admin_sidebar.php

add_hook("AdminClientAreaSidebar", 1, function(array $params) {
    // Add widget to client sidebar
    $panel = $params["sidebar"]->addPanel(
        "Custom Widgets",
        ["priority" => 50] // Lower = higher position
    );
    
    $panel->addItem(
        '<div class="custom-widget">
            <h4>Quick Stats</h4>
            <ul>
                <li>Services: ' . countClientServices($params["userid"]) . '</li>
                <li>Invoices: ' . countClientInvoices($params["userid"]) . '</li>
                <li>Tickets: ' . countClientTickets($params["userid"]) . '</li>
            </ul>
        </div>'
    );
    
    return $params;
});

function countClientServices(int $userId): int
{
    return Capsule::table("tblhosting")
        ->where("userid", $userId)
        ->count();
}
```

### Sidebar Panels

```php
add_hook("AdminClientAreaSidebar", 1, function(array $params) {
    $sidebar = $params["sidebar"];
    
    // Add custom actions panel
    $actionsPanel = $sidebar->addPanel("Client Actions");
    
    $actionsPanel->addItem(
        '<a href="clientnotes.php?userid=' . $params["userid"] . '" class="btn btn-default btn-block">
            <i class="fa fa-sticky-note"></i> Add Note
        </a>'
    );
    
    $actionsPanel->addItem(
        '<a href="clientservices.php?userid=' . $params["userid"] . '&action=create" class="btn btn-default btn-block">
            <i class="fa fa-plus"></i> Create Service
        </a>'
    );
    
    // Add info panel
    $infoPanel = $sidebar->addPanel("Account Info");
    
    $client = Capsule::table("tblclients")
        ->where("id", $params["userid"])
        ->first();
    
    $infoPanel->addItem('<strong>Member Since:</strong> ' . $client->datecreated);
    $infoPanel->addItem('<strong>Last Login:</strong> ' . ($client->lastlogin ?? 'Never'));
    $infoPanel->addItem('<strong>Status:</strong> ' . $client->status);
    
    return $params;
});
```

### Collapsible Sections

```php
add_hook("AdminClientAreaSidebar", 1, function(array $params) {
    $sidebar = $params["sidebar"];
    
    // Add collapsible panel
    $panel = $sidebar->addCollapsiblePanel("Recent Activity");
    
    $recentActivity = Capsule::table("mod_activity_log")
        ->where("user_id", $params["userid"])
        ->orderBy("created_at", "desc")
        ->limit(5)
        ->get();
    
    $activityHtml = '<ul class="activity-list">';
    foreach ($recentActivity as $activity) {
        $activityHtml .= '<li>' .
            '<span class="activity-icon fa fa-' . getActivityIcon($activity->action) . '"></span>' .
            '<span class="activity-text">' . $activity->action . '</span>' .
            '<span class="activity-time">' . $activity->created_at . '</span>' .
        '</li>';
    }
    $activityHtml .= '</ul>';
    
    $panel->addItem($activityHtml);
    
    return $params;
});

function getActivityIcon(string $action): string
{
    $icons = [
        "login" => "sign-in",
        "logout" => "sign-out",
        "invoice_paid" => "check",
        "service_created" => "plus",
        "ticket_created" => "ticket"
    ];
    
    return $icons[$action] ?? "circle";
}
```

### Sidebar Widgets Template

```smarty
<!-- /admin/templates/sidebar_widget.tpl -->
<div class="sidebar-widget" id="sidebar-{$widget.id}">
    <div class="widget-header">
        <h4>
            {if $widget.icon}
                <i class="fa fa-{$widget.icon}"></i>
            {/if}
            {$widget.title}
        </h4>
        {if $widget.collapsible}
            <button class="btn-collapse" data-target="sidebar-{$widget.id}">
                <i class="fa fa-chevron-{if $widget.collapsed}down{else}up{/if}"></i>
            </button>
        {/if}
    </div>
    <div class="widget-body" {if $widget.collapsed}style="display:none"{/if}>
        {$widget.content}
    </div>
</div>

<script>
$('.btn-collapse').on('click', function() {
    var target = $(this).data('target');
    var body = $('#' + target).find('.widget-body');
    body.toggle();
    $(this).find('i').toggleClass('fa-chevron-up fa-chevron-down');
});
</script>
```

### Quick Links Widget

```php
add_hook("AdminClientAreaSidebar", 1, function(array $params) {
    $panel = $params["sidebar"]->addPanel("Quick Links");
    
    $panel->addItem(
        '<a href="sendmessage.php?userid=' . $params["userid"] . '">
            <i class="fa fa-envelope"></i> Send Email
        </a>'
    );
    
    $panel->addItem(
        '<a href="supporttickets.php?action=open&userid=' . $params["userid"] . '">
            <i class="fa fa-life-ring"></i> Open Ticket
        </a>'
    );
    
    $panel->addItem(
        '<a href="invoice.php?action=create&userid=' . $params["userid"] . '">
            <i class="fa fa-file-invoice-dollar"></i> Create Invoice
        </a>'
    );
    
    return $params;
});
```

## Best Practices

1. **Priority**: Use priority to control panel order
2. **Collapsible**: Allow users to collapse panels
3. **Performance**: Minimize queries in sidebar
4. **Styling**: Match WHMCS admin theme
5. **Accessibility**: Support keyboard navigation
6. **Context**: Provide relevant context-specific actions
7. **User Preferences**: Save collapse/expand state
8. **Responsive**: Work on tablet view
