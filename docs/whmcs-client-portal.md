# WHMCS Client Portal

## Overview

The client portal in WHMCS provides customers with a web interface to manage their account, services, invoices, support tickets, and settings. The portal is customizable and can be themed to match your brand.

## Portal Access

### Login URL

```
https://yourdomain.com/clientarea.php
```

### Navigation Structure

```php
// Main navigation
[
    ['label' => 'Home', 'url' => '/clientarea.php'],
    ['label' => 'Services', 'url' => '/clientarea.php?action=services'],
    ['label' => 'Domains', 'url' => '/clientarea.php?action=domains'],
    ['label' => 'Billing', 'url' => '/clientarea.php?action=invoices'],
    ['label' => 'Support', 'url' => '/supporttickets.php'],
    ['label' => 'Account', 'url' => '/clientarea.php?action=account']
]
```

## Dashboard

### Client Dashboard

**Client Area > Dashboard**

```php
// Dashboard widgets
[
    'welcome' => [
        'title' => 'Welcome back, {$client_name}',
        'show_notifications' => true
    ],
    'services_summary' => [
        'active' => 5,
        'suspended' => 1,
        'total_monthly' => 150.00
    ],
    'invoices' => [
        'due_balance' => 100.00,
        'due_invoices' => 2
    ],
    'tickets' => [
        'open_tickets' => 3,
        'awaiting_reply' => 1
    ],
    'quick_actions' => [
        'order_new',
        'pay_invoice',
        'submit_ticket',
        'update_profile'
    ]
]
```

### Custom Dashboard

```php
// Add custom widgets
add_hook('ClientAreaHomepage', 1, function($vars) {
    return [
        'template' => 'custom-widget',
        'data' => ['custom' => 'data']
    ];
});
```

## Service Management

### Service List

**Client Area > Services > My Services**

```php
// Service display
[
    ['name' => 'Web Hosting - Basic', 'status' => 'Active', 'next_due' => '2024-06-01'],
    ['name' => 'SSL Certificate', 'status' => 'Active', 'next_due' => '2024-08-15'],
    ['name' => 'Email Hosting', 'status' => 'Suspended', 'next_due' => '2024-05-01']
]
```

### Service Details

**Client Area > Services > View Service**

```php
// Service detail page
[
    'product' => 'Web Hosting - Basic',
    'billing_cycle' => 'Monthly',
    'next_due_date' => '2024-06-01',
    'amount' => 10.00,
    'status' => 'Active',
    'custom_fields' => [...],
    'usage' => [
        'disk' => '5GB / 10GB',
        'bandwidth' => '100GB / 500GB'
    ]
]
```

### Service Actions

```php
// Available actions
[
    'upgrade' => true,
    'downgrade' => true,
    'cancel' => true,
    'view_details' => true,
    'download_invoice' => true
]
```

## Domain Management

### Domain List

**Client Area > Domains > My Domains**

```php
// Domain display
[
    ['domain' => 'example.com', 'status' => 'Active', 'expiry' => '2025-05-01', 'autorenew' => true],
    ['domain' => 'test.com', 'status' => 'Pending Transfer', 'expiry' => '2024-06-15']
]
```

### Domain Details

**Client Area > Domains > Manage Domain**

```php
// Domain management options
[
    'nameservers' => ['ns1.example.com', 'ns2.example.com'],
    'dns_management' => true,
    'email_forwarding' => true,
    'id_protection' => false,
    'transfer_lock' => true
]
```

## Invoice Management

### Invoice List

**Client Area > Billing > My Invoices**

```php
// Invoice list
[
    ['id' => '5678', 'date' => '2024-05-01', 'amount' => 100.00, 'status' => 'Paid'],
    ['id' => '5679', 'date' => '2024-06-01', 'amount' => 100.00, 'status' => 'Pending']
]
```

### Invoice View/Pay

**Client Area > Billing > View Invoice**

```php
// Invoice page
[
    'invoice_number' => 'INV-5678',
    'amount' => 100.00,
    'due_date' => '2024-06-01',
    'status' => 'Pending',
    'payment_methods' => ['credit_card', 'paypal', 'bank_transfer'],
    'line_items' => [...]
]
```

## Support Tickets

### Ticket List

**Client Area > Support > Support Tickets**

```php
// Ticket list
[
    ['id' => '1234', 'subject' => 'Login issue', 'status' => 'Open', 'last_reply' => '2024-05-15'],
    ['id' => '1235', 'subject' => 'Billing question', 'status' => 'Answered']
]
```

### Submit Ticket

**Client Area > Support > Open New Ticket**

```php
// Ticket form
[
    'department' => 'Technical Support',
    'subject' => 'required',
    'priority' => ['low', 'medium', 'high'],
    'attachment_allowed' => true,
    'max_attachments' => 5
]
```

## Account Settings

### Profile Management

**Client Area > Account > Edit Details**

```php
// Profile fields
[
    'first_name' => 'John',
    'last_name' => 'Doe',
    'email' => 'john@example.com',
    'phone' => '+1-555-0100',
    'company' => 'Example Inc'
]
```

### Change Password

**Client Area > Account > Security**

```php
// Password change form
[
    'current_password' => 'required',
    'new_password' => 'required',
    'confirm_password' => 'required'
]
```

### Payment Methods

**Client Area > Account > Payment Methods**

```php
// Saved payment methods
[
    ['type' => 'card', 'last4' => '4242', 'brand' => 'Visa', 'default' => true],
    ['type' => 'card', 'last4' => '5555', 'brand' => 'Mastercard']
]
```

## Portal Customization

### Theme Files

```
/whmcs/templates/clientarea/
├── default/
│   ├── header.tpl
│   ├── footer.tpl
│   ├── home.tpl
│   ├── sidebar.tpl
│   └── ...
└── custom-theme/
    └── (override files)
```

### Custom Pages

```php
// Add custom page
// File: /whmcs/templates/clientarea/custom-page.tpl

<div class="container">
    <h1>{$page_title}</h1>
    {$custom_content}
</div>

// Route: clientarea.php?action=custom-page
```

### Custom CSS

```css
/* custom.css */
.client-area-header {
    background-color: #3498db;
}

.btn-primary {
    background-color: #2ecc71;
}

.invoice-table {
    border: 1px solid #ddd;
}
```

## Portal Permissions

### Access Control

```php
// Control what clients see
[
    'allow_view_services' => true,
    'allow_view_invoices' => true,
    'allow_view_domains' => true,
    'allow_view_tickets' => true,
    'allow_download_invoices' => true,
    'allow_manage_contacts' => true
]
```

### Module-Level Permissions

```php
// Restrict service module access
[
    'product_id' => 1,
    'client_area_access' => 'full',
    'show_in_client_area' => true
]
```

## API Integration

```php
// Get client area data
$result = localAPI('GetClientDetails', [
    'clientid' => 123
]);

// Get client services
$result = localAPI('GetClientsProducts', [
    'clientid' => 123
]);

// Get client invoices
$result = localAPI('GetInvoices', [
    'userid' => 123
]);
```

## Best Practices

1. **Clear navigation**: Make it easy to find features
2. **Mobile responsive**: Ensure works on all devices
3. **Quick actions**: Add shortcuts to common tasks
4. **Regular updates**: Keep portal modern and secure
5. **Custom branding**: Match your company identity

## Related Documentation

- [Client Authentication](./whmcs-client-authentication.md)
- [Two-Factor Auth](./whmcs-two-factor-auth.md)
- [Invoice Templates](./whmcs-invoice-templates.md)
- [Template Customization](./whmcs-template-customization.md)