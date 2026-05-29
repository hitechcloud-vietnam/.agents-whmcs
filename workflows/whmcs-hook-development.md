# WHMCS Hook Development Workflow

## Purpose
Create custom hooks to extend WHMCS functionality

## Prerequisites
- WHMCS installed
- PHP development knowledge
- Understanding of WHMCS lifecycle

## Step 1: Understand Hook System

Hooks allow you to execute code at specific points:
- Before/After actions
- Output manipulation
- Data validation
- Custom notifications

## Step 2: Create Hooks Directory

```bash
mkdir -p /var/www/whmcs/custom/hooks
```

## Step 3: Create Basic Hook File

```php
<?php
// /var/www/whmcs/custom/hooks/example_hook.php

use WHMCS\Database\Capsule;

// Hook function
add_hook('AfterCronJob', 1, function($vars) {
    logActivity('Cron job completed at ' . date('Y-m-d H:i:s'));
});
```

## Step 4: Common Hook Points

### Client Hooks
```php
// After client registration
add_hook('ClientAreaRegistrationCompleted', 1, function($vars) {
    $clientId = $vars['userId'];
    // Send welcome email, create record, etc.
});

// Before client login
add_hook('ClientLogin', 1, function($vars) {
    $username = $vars['username'];
    // Log login attempt, check restrictions
});

// After client logout
add_hook('ClientLogout', 1, function($vars) {
    // Clear session data
});
```

### Order Hooks
```php
// After order placement
add_hook('OrderPaid', 1, function($vars) {
    $orderId = $vars['orderId'];
    // Send notification, update inventory
});

// Before order fulfillment
add_hook('OrderFulfillment', 1, function($vars) {
    $orderId = $vars['orderId'];
    // Custom provisioning logic
});
```

### Invoice Hooks
```php
// After invoice creation
add_hook('InvoiceCreated', 1, function($vars) {
    $invoiceId = $vars['invoiceId'];
    // Add custom line items
});

// After invoice paid
add_hook('InvoicePaid', 1, function($vars) {
    $invoiceId = $vars['invoiceId'];
    // Update CRM, trigger workflows
});
```

### Ticket Hooks
```php
// After ticket reply
add_hook('TicketReply', 1, function($vars) {
    $ticketId = $vars['ticketId'];
    // Notify external systems
});

// Before ticket close
add_hook('TicketClose', 1, function($vars) {
    // Validate ticket requirements
});
```

## Step 5: Output Manipulation Hooks

### Add CSS/JS to Client Area
```php
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    return '<link rel="stylesheet" href="custom/style.css">';
});

add_hook('ClientAreaFooterOutput', 1, function($vars) {
    return '<script src="custom/script.js"></script>';
});
```

### Modify Page Content
```php
add_hook('ClientAreaHomepagePanels', 1, function($vars) {
    return [
        'name' => 'custom_panel',
        'title' => 'Custom Information',
        'content' => '<p>Custom content here</p>',
    ];
});
```

## Step 6: Validation Hooks

```php
// Validate order
add_hook('PreDomainRegister', 1, function($vars) {
    $domain = $vars['sld'] . '.' . $vars['tld'];
    if (isPremiumDomain($domain)) {
        return ['error' => 'Premium domains require manual processing'];
    }
});

// Validate checkout
add_hook('ShoppingCartCheckoutValidation', 1, function($vars) {
    $cart = $vars['cart'];
    if ($cart['total'] > 10000) {
        return ['error' => 'Orders over $10,000 require approval'];
    }
});
```

## Step 7: Email Hooks

```php
// Modify email before send
add_hook('EmailPreSend', 1, function($vars) {
    $vars['subject'] = '[Company] ' . $vars['subject'];
    return $vars;
});

// After email sent
add_hook('EmailSent', 1, function($vars) {
    logEmailSent($vars);
});
```

## Step 8: API Hooks

```php
// Before API call
add_hook('PreAPICommand', 1, function($vars) {
    $action = $vars['action'];
    if ($action == 'GetClients' && !isAdmin()) {
        return ['error' => 'Unauthorized'];
    }
});
```

## Step 9: Database Operations

```php
use WHMCS\Database\Capsule;

add_hook('AfterCronJob', 1, function($vars) {
    // Insert record
    Capsule::table('custom_table')->insert([
        'last_run' => date('Y-m-d H:i:s'),
    ]);
    
    // Query data
    $results = Capsule::table('custom_table')
        ->where('status', 'active')
        ->get();
});
```

## Step 10: Admin Area Hooks

```php
// Add admin menu item
add_hook('AdminAreaHeader', 1, function($vars) {
    return '<div class="custom-admin-header">Custom Header</div>';
});

// Admin widget
add_hook('AdminHomeWidgets', 1, function($vars) {
    return [
        'title' => 'Custom Stats',
        'content' => 'Widget content here',
    ];
});
```

## Step 11: Third-Party Integration Hooks

### Slack Integration
```php
add_hook('TicketOpen', 1, function($vars) {
    sendSlackNotification([
        'ticket_id' => $vars['ticketId'],
        'subject' => $vars['subject'],
        'priority' => $vars['priority'],
    ]);
});
```

### CRM Integration
```php
add_hook('InvoicePaid', 1, function($vars) {
    $invoice = localAPI('GetInvoice', ['invoiceid' => $vars['invoiceId']]);
    updateCRMLead($invoice);
});
```

## Step 12: Hook Priority

Hooks execute in priority order (lower = earlier):

```php
add_hook('InvoicePaid', 1, function($vars) {
    // Executes first
});

add_hook('InvoicePaid', 10, function($vars) {
    // Executes second
});

add_hook('InvoicePaid', 100, function($vars) {
    // Executes last
});
```

## Step 13: Return Values

```php
// Return error to prevent action
add_hook('PreDomainRegister', 1, function($vars) {
    if ($blocked) {
        return [
            'error' => 'Domain registration blocked',
        ];
    }
});

// Return array to modify output
add_hook('ClientAreaPage', 1, function($vars) {
    return [
        'additionalVar' => 'value',
    ];
});
```

## Step 14: Debug Hooks

```php
add_hook('InvoicePaid', 1, function($vars) {
    logActivity('InvoicePaid Hook Debug: ' . print_r($vars, true));
});
```

## Hook Development Checklist

- [ ] Hooks directory created
- [ ] Basic hook file created
- [ ] Client hooks implemented
- [ ] Order hooks implemented
- [ ] Invoice hooks implemented
- [ ] Ticket hooks implemented
- [ ] Output hooks implemented
- [ ] Validation hooks implemented
- [ ] Database operations tested
- [ ] Third-party integrations added
- [ ] Debug logging configured
