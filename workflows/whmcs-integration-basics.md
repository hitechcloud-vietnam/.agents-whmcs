# WHMCS Integration Basics Workflow

## Purpose
Integrate WHMCS with external services and applications

## Prerequisites
- WHMCS installed
- API access or module files
- Third-party service credentials

## Step 1: Identify Integration Points

Common integration types:
- Payment gateways
- Domain registrars
- Server provisioning
- CRM systems
- Email marketing
- Analytics
- Support tools

## Step 2: Payment Gateway Integration

### Standard Gateway
```bash
# Upload gateway to modules
cp gateway_module /var/www/whmcs/modules/gateways/
```

### Configure Gateway
Navigate to: Setup > Payments > Payment Gateways

Activate and configure gateway credentials.

## Step 3: Server Provisioning Integration

### Install Server Module
```bash
cp server_module /var/www/whmcs/modules/servers/
```

### Configure Server
Navigate to: Setup > Servers > Create New Server

```
Name: Production Server
Hostname: server1.yourdomain.com
IP Address: 192.168.1.1
Module: Your Server Module
```

## Step 4: CRM Integration

### Create CRM Sync Module

```php
<?php
// hooks.php

add_hook('ClientLogin', 1, function($vars) {
    syncClientToCRM($vars['userId']);
});

add_hook('InvoicePaid', 1, function($vars) {
    updateCRMInvoice($vars['invoiceId']);
});
```

### CRM Integration Points
- Sync client data on registration
- Update CRM on payment
- Sync product purchases
- Push support tickets

## Step 5: Email Marketing Integration

```php
<?php
// hooks.php

add_hook('ClientRegistrationCompleted', 1, function($vars) {
    addToEmailList($vars['email'], [
        'name' => $vars['firstname'] . ' ' . $vars['lastname'],
    ]);
});

add_hook('OrderPaid', 1, function($vars) {
    trackPurchase($vars['email'], $vars['orderId']);
});
```

### Email Platforms
- Mailchimp
- SendGrid
- ConvertKit
- ActiveCampaign

## Step 6: Analytics Integration

### Google Analytics
```php
<?php
// hooks.php

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $script = "
    gtag('config', 'UA-XXXXXXXXX-X', {
        'user_id': '" . $vars['user']->id . "'
    });
    ";
    return '<script>' . $script . '</script>';
});
```

### Facebook Pixel
```php
add_hook('OrderPaid', 1, function($vars) {
    $order = getOrderDetails($vars['orderId']);
    echo "<script>
    fbq('track', 'Purchase', {
        value: " . $order['total'] . ",
        currency: 'USD'
    });
    </script>";
});
```

## Step 7: Support Tool Integration

### Help Scout
```php
add_hook('TicketOpened', 1, function($vars) {
    createHelpScoutConversation($vars['ticketId']);
});
```

### Freshdesk
```php
add_hook('TicketOpened', 1, function($vars) {
    createFreshdeskTicket($vars['ticketId']);
});
```

## Step 8: Accounting Integration

### QuickBooks/Xero
```php
add_hook('InvoicePaid', 1, function($vars) {
    $invoice = localAPI('GetInvoice', ['invoiceid' => $vars['invoiceId']]);
    createAccountingEntry($invoice);
});
```

## Step 9: Domain Registrar Integration

### Configure Registrar
Navigate to: Setup > Products/Services > Domain Registrars

1. Select registrar module
2. Enter API credentials
3. Configure settings

## Step 10: SSL Certificate Integration

### MarketConnect
Navigate to: Setup > MarketConnect

1. Select SSL provider
2. Configure pricing
3. Enable auto-provisioning

## Integration Checklist

- [ ] Integration points identified
- [ ] Payment gateway integrated
- [ ] Server provisioning integrated
- [ ] CRM integrated
- [ ] Email marketing integrated
- [ ] Analytics integrated
- [ ] Support tools integrated
- [ ] Accounting integrated
- [ ] Registrar integrated
- [ ] SSL integration configured
