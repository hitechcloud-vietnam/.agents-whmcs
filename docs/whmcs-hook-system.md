# WHMCS Hook System Reference

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-hooks-development`, `module-hook-reference`

---

## Overview

The WHMCS Hook System allows developers to execute custom code at specific points during WHMCS operations. Hooks are the primary mechanism for extending WHMCS functionality without modifying core files.

---

## Hook Registration

### Basic Syntax

```php
<?php
/**
 * Register a hook
 *
 * @param string $hookName    The hook point to attach to
 * @param int    $priority    Execution priority (lower = earlier)
 * @param closure $callback    The function to execute
 * @return bool
 */
add_hook('HookName', $priority, function($vars) {
    // Your hook code here
    return $vars;
});
```

### Priority System

- Priority 1-10: Early execution (core operations)
- Priority 11-50: Standard execution
- Priority 51-100: Late execution (after other hooks)
- Multiple hooks with same priority execute in registration order

---

## Client Hooks

### Registration Hooks

| Hook Name | Parameters | Description |
|-----------|------------|-------------|
| `ClientAdd` | `userid`, `firstname`, `lastname`, `email` | New client registered |
| `PreClientAdd` | `params` | Before client creation (can modify data) |
| `ClientEdit` | `userid`, `params` | Client profile updated |
| `ClientDelete` | `userid` | Client deleted |
| `ClientChangePassword` | `userid`, `password` | Client password changed |

```php
// Log new client registration
add_hook('ClientAdd', 1, function($vars) {
    logActivity("New client registered: " . $vars['email']);

    Capsule::table('mod_audit_log')->insert([
        'action' => 'client_registered',
        'user_id' => $vars['userid'],
        'email' => $vars['email'],
        'created_at' => date('Y-m-d H:i:s'),
    ]);
});

// Validate client before creation
add_hook('PreClientAdd', 1, function($vars) {
    // Block specific email domains
    $blockedDomains = ['tempmail.com', 'throwaway.net'];

    foreach ($blockedDomains as $domain) {
        if (strpos($vars['email'], '@' . $domain) !== false) {
            return ['error' => 'Email domain not allowed'];
        }
    }

    return $vars;
});
```

### Authentication Hooks

| Hook Name | Parameters | Description |
|-----------|------------|-------------|
| `ClientLogin` | `userid` | Client logged in |
| `ClientLogout` | `userid` | Client logged out |
| `PreLoginClient` | `username` | Before login validation |
| `LoginVerify` | `username`, `password`, `result` | After login verification |
| `TwoFactorAuthentication` | `userid`, `code`, `success` | Two-factor verification |

```php
// Track client login with IP
add_hook('ClientLogin', 1, function($vars) {
    Capsule::table('mod_login_tracking')->insert([
        'user_id' => $vars['userid'],
        'ip_address' => $_SERVER['REMOTE_ADDR'],
        'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
        'login_time' => date('Y-m-d H:i:s'),
    ]);
});

// Block login from specific IPs
add_hook('PreLoginClient', 1, function($vars) {
    $blockedIPs = ['192.168.1.100', '10.0.0.50'];

    if (in_array($_SERVER['REMOTE_ADDR'], $blockedIPs)) {
        return ['error' => 'Access denied'];
    }
});
```

---

## Invoice Hooks

| Hook Name | Parameters | Description |
|-----------|------------|-------------|
| `InvoiceCreation` | `invoiceid`, `userid` | Invoice created |
| `InvoicePaid` | `invoiceid`, `userid`, `amount` | Invoice payment received |
| `InvoiceCancelled` | `invoiceid` | Invoice cancelled |
| `InvoiceRefunded` | `invoiceid` | Invoice refunded |
| `InvoicePrePayment` | `invoiceid` | Before payment processing |
| `InvoiceCreationPreTax` | `invoiceid` | Before tax calculation |

```php
// Send notification on payment
add_hook('InvoicePaid', 1, function($vars) {
    $invoice = local_api('GetInvoice', ['invoice_id' => $vars['invoiceid']]);

    // Send to Slack/Discord webhook
    sendNotification([
        'text' => "Payment received: $" . $vars['amount'] .
                  " from User #" . $vars['userid']
    ]);
});

// Auto-apply credit on invoice creation
add_hook('InvoiceCreation', 1, function($vars) {
    $clientCredits = Capsule::table('tblcredit')
        ->where('clientid', $vars['userid'])
        ->sum('amount');

    if ($clientCredits > 0) {
        // Apply credit to invoice
        local_api('ApplyCredit', [
            'client_id' => $vars['userid'],
            'invoice_id' => $vars['invoiceid'],
            'amount' => min($clientCredits, $invoiceAmount),
        ]);
    }
});
```

---

## Service Hooks

| Hook Name | Parameters | Description |
|-----------|------------|-------------|
| `AfterModuleCreate` | `serviceid`, `userid` | Service provisioned |
| `AfterModuleSuspend` | `serviceid`, `userid` | Service suspended |
| `AfterModuleUnsuspend` | `serviceid`, `userid` | Service reactivated |
| `AfterModuleTerminate` | `serviceid`, `userid` | Service terminated |
| `AfterModuleChangePackage` | `serviceid`, `params` | Package changed |
| `AfterModuleChangePassword` | `serviceid`, `newpassword` | Password changed |
| `PreServiceDelete` | `userid`, `serviceid` | Before service deletion |
| `ServiceEdit` | `serviceid`, `params` | Service edited |

```php
// Send welcome email on service creation
add_hook('AfterModuleCreate', 1, function($vars) {
    $service = Capsule::table('tblhosting')
        ->where('id', $vars['serviceid'])
        ->first();

    send_email([
        'type' => 'product',
        'id' => $service->packageid,
        'customvars' => [
            'service_id' => $vars['serviceid'],
            'service_domain' => $service->domain,
            'server_ip' => $service->dedicatedip,
        ],
    ], $vars['userid']);
});

// Sync service data to external system
add_hook('AfterModuleCreate', 50, function($vars) {
    $api = new ExternalApiClient();
    $api->createServer([
        'user_id' => $vars['userid'],
        'service_id' => $vars['serviceid'],
    ]);
});
```

---

## Domain Hooks

| Hook Name | Parameters | Description |
|-----------|------------|-------------|
| `DomainRegister` | `domainid` | Domain registered |
| `DomainTransferCompleted` | `domainid` | Transfer completed |
| `DomainTransferFailed` | `domainid` | Transfer failed |
| `DomainRenew` | `domainid` | Domain renewed |
| `DomainDeletion` | `domainid` | Domain deleted |
| `DomainPreRegistration` | `domain`, `params` | Before domain registration |

```php
// Update DNS on domain registration
add_hook('DomainRegister', 1, function($vars) {
    $domain = Capsule::table('tbldomains')
        ->where('id', $vars['domainid'])
        ->first();

    // Configure DNS records
    updateDnsRecords([
        'domain' => $domain->domain,
        'action' => 'create',
    ]);
});
```

---

## Order Hooks

| Hook Name | Parameters | Description |
|-----------|------------|-------------|
| `OrderAccepted` | `orderid` | Order accepted |
| `OrderPaid` | `orderid`, `userid` | Order payment received |
| `OrderCancelled` | `orderid` | Order cancelled |
| `OrderFraud` | `orderid` | Order flagged as fraud |
| `OrderProductValidation` | `params` | Product validation |
| `OrderAdditionalTransaction` | `orderid`, `amount`, `transaction` | Additional payment |

```php
// Check fraud on new order
add_hook('OrderProductValidation', 1, function($vars) {
    $ip = $_SERVER['REMOTE_ADDR'];
    $recentOrders = Capsule::table('tblorders')
        ->where('ipaddress', $ip)
        ->whereRaw("date > DATE_SUB(NOW(), INTERVAL 24 HOUR)")
        ->count();

    if ($recentOrders >= 3) {
        return ['error' => 'Too many orders from this IP'];
    }

    return $vars;
});

// Mark high-value orders for review
add_hook('OrderPaid', 1, function($vars) {
    $order = Capsule::table('tblorders')
        ->where('id', $vars['orderid'])
        ->first();

    if ($order->total >= 1000) {
        Capsule::table('tblorders')
            ->where('id', $vars['orderid'])
            ->update(['notes' => 'High value order - manual review required']);
    }
});
```

---

## Ticket Hooks

| Hook Name | Parameters | Description |
|-----------|------------|-------------|
| `TicketOpen` | `ticketid`, `userid`, `deptid` | New ticket created |
| `TicketReply` | `ticketid`, `userid` | Ticket replied |
| `TicketClose` | `ticketid` | Ticket closed |
| `TicketAddNote` | `ticketid` | Note added |
| `TicketOpenPre` | `params` | Before ticket creation |
| `TicketEscalate` | `ticketid`, `level` | Ticket escalated |

```php
// Auto-assign tickets based on keywords
add_hook('TicketOpen', 1, function($vars) {
    $ticket = Capsule::table('tbltickets')
        ->where('id', $vars['ticketid'])
        ->first();

    $keywords = [
        'billing' => 2,
        'refund' => 3,
        'technical' => 4,
        'sales' => 5,
    ];

    foreach ($keywords as $keyword => $deptId) {
        if (stripos($ticket->title, $keyword) !== false) {
            Capsule::table('tbltickets')
                ->where('id', $vars['ticketid'])
                ->update(['did' => $deptId]);
            break;
        }
    }
});
```

---

## Cron Hooks

| Hook Name | Parameters | Description |
|-----------|------------|-------------|
| `DailyCronJob` | - | Daily cron execution |
| `HourlyCronJob` | - | Hourly cron execution |
| `AutoTaxProvision` | - | Auto tax calculation |
| `LicensingCheck` | - | License verification |

```php
// Custom daily maintenance
add_hook('DailyCronJob', 1, function($vars) {
    // Clean up old logs
    Capsule::table('mod_temp_data')
        ->where('created_at', '<', date('Y-m-d H:i:s', strtotime('-30 days')))
        ->delete();

    // Sync with external systems
    $api = new ExternalSync();
    $api->syncData();
});

// Generate usage reports
add_hook('DailyCronJob', 50, function($vars) {
    $usages = Capsule::table('tblhosting')
        ->selectRaw('packageid, COUNT(*) as count')
        ->groupBy('packageid')
        ->get();

    foreach ($usages as $usage) {
        Capsule::table('mod_usage_reports')->insert([
            'report_date' => date('Y-m-d'),
            'package_id' => $usage->packageid,
            'count' => $usage->count,
        ]);
    }
});
```

---

## Admin Area Hooks

| Hook Name | Parameters | Description |
|-----------|------------|-------------|
| `AdminAreaPages` | (dynamic) | Add admin dashboard pages |
| `AdminAreaWidget` | (dynamic) | Add dashboard widgets |
| `AdminAreaNavItems` | (dynamic) | Add navigation items |
| `AdminAreaHeaderOutput` | - | Add header HTML |
| `AdminAreaFooterOutput` | - | Add footer HTML |
| `AdminPrintViewBinary` | - | Modify print views |
| `AdminAreaPageContext` | (dynamic) | Modify page context |

```php
// Add admin navigation item
add_hook('AdminAreaNavItems', 1, function($vars) {
    return [
        'uri' => 'addon_module.php?module=my_module',
        'label' => 'My Module',
        'icon' => 'fa-cog',
        'order' => 100,
    ];
});

// Add CSS to admin area
add_hook('AdminAreaHeaderOutput', 1, function($vars) {
    return '<style>
        .custom-admin-style {
            border: 2px solid #ff6600;
        }
    </style>';
});
```

---

## Client Area Hooks

| Hook Name | Parameters | Description |
|-----------|------------|-------------|
| `ClientAreaHeaderOutput` | - | Add header HTML |
| `ClientAreaFooterOutput` | - | Add footer HTML |
| `ClientAreaPage` | (dynamic) | Modify client pages |
| `ClientAreaSidebars` | (dynamic) | Add sidebars |
| `PredefinedSmartyVariables` | (dynamic) | Add template variables |
| `ClientAreaMyAccount` | - | My Account page |

```php
// Add custom CSS to client area
add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    return '<style>
        body.custom-theme {
            background-color: #f5f5f5;
        }
    </style>';
});

// Add custom template variables
add_hook('PredefinedSmartyVariables', 1, function($vars) {
    return [
        'custom_variable' => 'My Custom Value',
        'module_status' => Capsule::table('mod_my_table')
            ->where('user_id', $_SESSION['uid'])
            ->value('status'),
    ];
});
```

---

## Output Hooks

### ViewInvoice Output

```php
add_hook('ViewInvoiceOutput', 1, function($vars) {
    // Add custom content to invoice
    return '<div class="invoice-custom-content">
        <p>Thank you for your business!</p>
    </div>';
});
```

### Email Hooks

```php
// Modify email before sending
add_hook('EmailPreSend', 1, function($vars) {
    // Add tracking pixel
    $vars['body'] .= '<img src="https://example.com/track/' . $vars['id'] . '">';
    return $vars;
});
```

---

## Hook Parameters Reference

### Common Variables Available

| Context | Available Variables |
|---------|-------------------|
| Client hooks | `userid`, `firstname`, `lastname`, `email`, `companyname` |
| Invoice hooks | `invoiceid`, `userid`, `amount`, `status` |
| Service hooks | `serviceid`, `userid`, `params` |
| Domain hooks | `domainid`, `domain`, `#params` |
| Order hooks | `orderid`, `userid`, `total`, `products` |
| Ticket hooks | `ticketid`, `userid`, `deptid`, `subject` |

---

## Best Practices

1. **Use unique hook names** - Prefix with module name to avoid conflicts
2. **Set appropriate priority** - Lower for critical checks, higher for notifications
3. **Return errors properly** - Return `['error' => 'message']` to block operations
4. **Log hook execution** - For debugging in development
5. **Handle missing data** - Check for null/undefined variables
6. **Sanitize input** - Always validate and sanitize hook parameters
7. **Avoid heavy operations** - Keep hook execution fast

---

## Related Documentation

- [Hooks Reference](hooks-reference.md)
- [Module Hook Reference](module-hook-reference.md)
- [API Endpoints Reference](api-endpoints-reference.md)
