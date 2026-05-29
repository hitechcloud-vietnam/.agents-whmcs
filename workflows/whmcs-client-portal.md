# WHMCS Client Portal Customization Workflow

## Overview
Customize the client area portal with branding, widgets, and enhanced functionality.

## Prerequisites
- WHMCS v8.0+
- Template access

## Step-by-Step Guide

### Step 1: Client Area Hooks
```php
<?php
add_hook('ClientAreaHomePage', 1, function($vars) {
    return [
        'custom_widget' => [
            'title' => 'Quick Actions',
            'items' => [
                ['label' => 'Order New Service', 'link' => 'cart.php'],
                ['label' => 'Pay Invoice', 'link' => 'invoices.php'],
                ['label' => 'Support Ticket', 'link' => 'supporttickets.php'],
            ],
        ],
    ];
});
```

### Step 2: Custom Dashboard Widget
```php
<?php
add_hook('AdminHomeWidgets', 1, function() {
    return new class extends \WHMCS\Module\AbstractWidget {
        public function getTitle(): string
        {
            return 'Statistics';
        }
        
        public function getBody(): string
        {
            $totalClients = \WHMCS\Database\Capsule::table('tblclients')->count();
            return "<p>Total Clients: $totalClients</p>";
        }
    };
});
```

## Checklist
- Branding applied
- Custom widgets added
- Navigation customized
- Mobile responsive tested
