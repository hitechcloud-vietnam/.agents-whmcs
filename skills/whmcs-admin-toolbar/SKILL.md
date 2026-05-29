# WHMCS Admin Toolbar

## Overview
Guide for customizing the WHMCS admin toolbar. Covers toolbar buttons, actions, and contextual tools.

## Toolbar Customization

### Add Toolbar Button

```php
<?php
// /includes/hooks/admin_toolbar.php

add_hook("AdminClientAreaToolbar", 1, function(array $params) {
    $toolbar = $params["toolbar"];
    $userId = $params["userid"];
    
    // Add custom action button
    $toolbar->addButton(
        "custom_action",
        [
            "label" => "Custom Action",
            "icon" => "fa-magic",
            "class" => "btn-primary",
            "href" => "custom.php?userid=" . $userId,
            "onclick" => "return confirm('Are you sure?');"
        ]
    );
    
    // Add dropdown menu
    $toolbar->addDropdown(
        "more_actions",
        [
            "label" => "More Actions",
            "icon" => "fa-ellipsis-v"
        ]
    );
    
    $toolbar->addDropdownItem(
        "more_actions",
        [
            "label" => "Export Data",
            "icon" => "fa-download",
            "href" => "export.php?userid=" . $userId
        ]
    );
    
    $toolbar->addDropdownItem(
        "more_actions",
        [
            "label" => "Merge Account",
            "icon" => "fa-compress-arrows-alt",
            "href" => "merge.php?userid=" . $userId
        ]
    );
    
    return $params;
});
```

### Contextual Toolbar

```php
add_hook("AdminServiceToolbar", 1, function(array $params) {
    $serviceId = $params["serviceid"];
    $service = Capsule::table("tblhosting")->where("id", $serviceId)->first();
    
    $toolbar = $params["toolbar"];
    
    // Add service-specific actions
    if ($service->domainstatus === "Active") {
        $toolbar->addButton("suspend_service", [
            "label" => "Suspend",
            "icon" => "fa-pause",
            "class" => "btn-warning",
            "href" => "clientsservices.php?action=suspend&id=" . $serviceId
        ]);
        
        $toolbar->addButton("terminate_service", [
            "label" => "Terminate",
            "icon" => "fa-trash",
            "class" => "btn-danger",
            "href" => "clientsservices.php?action=terminate&id=" . $serviceId
        ]);
    }
    
    if ($service->domainstatus === "Suspended") {
        $toolbar->addButton("unsuspend_service", [
            "label" => "Unsuspend",
            "icon" => "fa-play",
            "class" => "btn-success",
            "href" => "clientsservices.php?action=unsuspend&id=" . $serviceId
        ]);
    }
    
    return $params;
});
```

### Batch Toolbar Actions

```php
add_hook("AdminBatchToolbar", 1, function(array $params) {
    $toolbar = $params["toolbar"];
    
    // Add batch action buttons
    $toolbar->addBatchAction("mass_email", [
        "label" => "Send Email",
        "icon" => "fa-envelope",
        "href" => "massmail.php?ids=" . implode(",", $params["selected_ids"])
    ]);
    
    $toolbar->addBatchAction("mass_export", [
        "label" => "Export Selected",
        "icon" => "fa-download",
        "href" => "export.php?ids=" . implode(",", $params["selected_ids"])
    ]);
    
    $toolbar->addBatchAction("mass_delete", [
        "label" => "Delete Selected",
        "icon" => "fa-trash",
        "class" => "btn-danger",
        "onclick" => "return confirm('Delete selected items?');",
        "href" => "delete.php?ids=" . implode(",", $params["selected_ids"])
    ]);
    
    return $params;
});
```

### Toolbar Template

```smarty
<!-- /admin/templates/toolbar.tpl -->
<div class="btn-toolbar" role="toolbar">
    <div class="btn-group">
        {foreach $toolbar.buttons as $button}
            <a href="{$button.href}" 
               class="btn {$button.class}"
               {if $button.onclick}onclick="{$button.onclick}"{/if}>
                {if $button.icon}
                    <i class="fa fa-{$button.icon}"></i>
                {/if}
                {$button.label}
            </a>
        {/foreach}
    </div>
    
    {if $toolbar.dropdowns}
        <div class="btn-group">
            {foreach $toolbar.dropdowns as $dropdown}
                <button type="button" 
                        class="btn btn-default dropdown-toggle" 
                        data-toggle="dropdown">
                    <i class="fa fa-{$dropdown.icon}"></i>
                    {$dropdown.label}
                    <span class="caret"></span>
                </button>
                <ul class="dropdown-menu">
                    {foreach $dropdown.items as $item}
                        <li>
                            <a href="{$item.href}">
                                <i class="fa fa-{$item.icon}"></i>
                                {$item.label}
                            </a>
                        </li>
                    {/foreach}
                </ul>
            {/foreach}
        </div>
    {/if}
</div>
```

## Best Practices

1. **Consistency**: Match WHMCS button styles
2. **Permissions**: Show buttons based on permissions
3. **Confirmation**: Add confirm dialogs for destructive actions
4. **Icons**: Use appropriate FontAwesome icons
5. **Grouping**: Group related actions together
6. **Mobile**: Ensure toolbar is usable on mobile
7. **Keyboard**: Support keyboard shortcuts
8. **Context**: Show relevant actions for context
