# WHMCS Navigation Menu Customization Workflow

## Overview
Customize WHMCS navigation menus for both client and admin areas.

## Prerequisites
- WHMCS v8.0+
- Template access

## Step-by-Step Guide

### Step 1: Client Navigation Hook
```php
<?php
add_hook('ClientAreaNav', 1, function() {
    return [
        'name' => 'My Services',
        'label' => 'My Services',
        'uri' => 'clientarea.php?action=services',
        'icon' => 'fa-server',
        'order' => 2,
    ];
});
```

### Step 2: Conditional Navigation
```php
<?php
add_hook('ClientAreaNav', 1, function() {
    if (!\Auth::check()) {
        return [
            [
                'name' => 'Login',
                'uri' => 'dologin.php',
                'icon' => 'fa-sign-in',
            ],
            [
                'name' => 'Register',
                'uri' => 'register.php',
                'icon' => 'fa-user-plus',
            ],
        ];
    }
    
    return [
        [
            'name' => 'Dashboard',
            'uri' => 'clientarea.php',
            'icon' => 'fa-home',
        ],
        [
            'name' => 'Services',
            'uri' => 'clientarea.php?action=services',
            'icon' => 'fa-server',
        ],
    ];
});
```

### Step 3: Admin Navigation
```php
<?php
add_hook('AdminAreaNav', 1, function() {
    return [
        [
            'name' => 'Module Settings',
            'label' => 'Module Settings',
            'uri' => 'addonmodules.php?module=yourmodule',
            'icon' => 'fa-cog',
        ],
    ];
});
```

## Checklist
- Client navigation customized
- Admin navigation enhanced
- Conditional items added
- Permissions set
