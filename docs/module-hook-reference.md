# WHMCS Module Hook Reference

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

This reference documents all available hook points for WHMCS modules, organized by category. Hooks allow modules to integrate with and extend WHMCS functionality at key points in the application lifecycle.

---

## Hook Registration

### Declaring Hooks in Addon Modules

```php
/**
 * Register module hooks
 */
function your_module_registerHooks()
{
    return [
        // Client hooks
        'ClientAdd'              => 'onClientAdd',
        'ClientEdit'             => 'onClientEdit',
        'ClientDelete'           => 'onClientDelete',
        'ClientLogin'            => 'onClientLogin',
        
        // Order hooks
        'AcceptOrder'            => 'onAcceptOrder',
        'CancelOrder'            => 'onCancelOrder',
        'PendingOrder'           => 'onPendingOrder',
        
        // Invoice hooks
        'InvoiceCreated'         => 'onInvoiceCreated',
        'InvoicePaid'            => 'onInvoicePaid',
        'InvoiceCancelled'       => 'onInvoiceCancelled',
        
        // Service hooks
        'ServiceCreate'          => 'onServiceCreate',
        'ServiceTerminate'       => 'onServiceTerminate',
        'ServiceRenew'           => 'onServiceRenew',
        
        // Admin hooks
        'AdminAreaHeader'        => 'onAdminAreaHeader',
        'AdminAreaPage'          => 'onAdminAreaPage',
    ];
}

/**
 * Register hook with priority
 */
function your_module_registerHooks()
{
    return [
        'ClientAdd' => [
            'handler'   => 'onClientAdd',
            'priority'  => 100,  // Lower = earlier execution
        ],
    ];
}
```

### Manual Hook Registration

```php
/**
 * Register hooks manually
 */
function your_module_registerHooks()
{
    $hooks = [
        'ClientAdd' => 'onClientAdd',
        'InvoicePaid' => 'onInvoicePaid',
    ];
    
    foreach ($hooks as $hookPoint => $function) {
        add_hook($hookPoint, 100, function($vars) use ($function) {
            return $function($vars);
        });
    }
}

// Or use add_hook() directly in module activation
add_hook('ClientAdd', 100, function($vars) {
    your_module_onClientAdd($vars);
});
```

## Client Hooks

### ClientAdd

```php
/**
 * New client registration
 * 
 * $vars['userid'] - New client ID
 * $vars['firstname'], $vars['lastname'], $vars['email']
 * $vars['companyname'], $vars['phonenumber']
 */
function onClientAdd($vars)
{
    $clientId = $vars['userid'];
    
    // Initialize module data for new client
    insert_query('mod_your_table', [
        'client_id'  => $clientId,
        'status'     => 'inactive',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    logActivity("Your module: Initialized for new client {$clientId}");
    
    return true;
}
```

### ClientEdit

```php
/**
 * Client data update
 * 
 * $vars['userid'] - Client ID
 * $vars['fields'] - Changed fields array
 */
function onClientEdit($vars)
{
    $clientId = $vars['userid'];
    
    // Check if email changed
    if (isset($vars['fields']['email'])) {
        // Sync email with external service
        your_module_syncEmail($clientId, $vars['fields']['email']);
    }
    
    return true;
}
```

### ClientDelete

```php
/**
 * Client deletion
 * 
 * $vars['userid'] - Client being deleted
 * $vars['email'], $vars['firstname'] - Client details
 */
function onClientDelete($vars)
{
    $clientId = $vars['userid'];
    
    // Clean up module data
    WHMCS\Database\Capsule::table('mod_your_table')
        ->where('client_id', $clientId)
        ->delete();
    
    // Notify external service
    your_module_notifyDeletion($clientId);
    
    return true;
}
```

### ClientLogin

```php
/**
 * Client login event
 * 
 * $vars['userid'] - User ID
 * $vars['redirect'] - Can be modified to redirect elsewhere
 */
function onClientLogin($vars)
{
    $clientId = $vars['userid'];
    
    // Track login
    insert_query('mod_your_logins', [
        'client_id'  => $clientId,
        'ip_address' => $_SERVER['REMOTE_ADDR'],
        'user_agent' => $_SERVER['HTTP_USER_AGENT'],
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    return true;
}
```

## Order Hooks

### AcceptOrder

```php
/**
 * Order accepted/activated
 * 
 * $vars['orderid'] - Order ID
 * $vars['userid'] - Client ID
 * $vars['products'] - Array of products in order
 */
function onAcceptOrder($vars)
{
    $orderId = $vars['orderid'];
    
    // Get order products
    $products = $vars['products'];
    
    foreach ($products as $product) {
        $serviceId = $product['serviceid'];
        
        // Activate module for this service
        your_module_activateService($serviceId);
    }
    
    return true;
}
```

### CancelOrder

```php
/**
 * Order cancelled
 * 
 * $vars['orderid'] - Order ID
 * $vars['userid'] - Client ID
 * $vars['cancellationrequested'] - Who requested
 */
function onCancelOrder($vars)
{
    $orderId = $vars['orderid'];
    
    // Process cancellation
    your_module_handleCancellation($orderId);
    
    return true;
}
```

### PendingOrder

```php
/**
 * Order moved to pending
 * 
 * $vars['orderid'] - Order ID
 * $vars['userid'] - Client ID
 * $vars['newstatus'] - Will be 'Pending'
 */
function onPendingOrder($vars)
{
    // Handle fraud check or manual review
    your_module_queueForReview($vars['orderid']);
    
    return true;
}
```

## Invoice Hooks

### InvoiceCreated

```php
/**
 * Invoice created
 * 
 * $vars['invoiceid'] - Invoice ID
 * $vars['userid'] - Client ID
 * $vars['total'] - Invoice total
 */
function onInvoiceCreated($vars)
{
    $invoiceId = $vars['invoiceid'];
    
    // Send additional notification
    your_module_notifyInvoiceCreated($invoiceId);
    
    return true;
}
```

### InvoicePaid

```php
/**
 * Invoice payment received
 * 
 * $vars['invoiceid'] - Invoice ID
 * $vars['userid'] - Client ID
 * $vars['total'] - Amount paid
 * $vars['paymentmethod'] - Payment gateway used
 * $vars['transid'] - Transaction ID
 */
function onInvoicePaid($vars)
{
    $invoiceId = $vars['invoiceid'];
    $clientId = $vars['userid'];
    $amount = $vars['total'];
    
    // Credit external account
    your_module_creditAccount($clientId, $amount, $invoiceId);
    
    // Trigger webhook
    your_module_triggerWebhook('payment.received', [
        'invoice_id' => $invoiceId,
        'client_id'  => $clientId,
        'amount'     => $amount,
    ]);
    
    return true;
}
```

### InvoiceCancelled

```php
/**
 * Invoice cancelled
 * 
 * $vars['invoiceid'] - Invoice ID
 * $vars['userid'] - Client ID
 */
function onInvoiceCancelled($vars)
{
    // Handle cancellation
    your_module_handleInvoiceCancellation($vars['invoiceid']);
    
    return true;
}
```

## Service Hooks

### ServiceCreate

```php
/**
 * New service created
 * 
 * $vars['serviceid'] - Service ID
 * $vars['userid'] - Client ID
 * $vars['pid'] - Product ID
 * $vars['domain'] - Service domain
 */
function onServiceCreate($vars)
{
    $serviceId = $vars['serviceid'];
    
    // Provision service in external system
    your_module_provisionService($serviceId);
    
    return true;
}
```

### ServiceTerminate

```php
/**
 * Service termination
 * 
 * $vars['serviceid'] - Service ID
 * $vars['userid'] - Client ID
 */
function onServiceTerminate($vars)
{
    $serviceId = $vars['serviceid'];
    
    // Terminate in external system
    your_module_terminateService($serviceId);
    
    return true;
}
```

### ServiceRenew

```php
/**
 * Service renewal
 * 
 * $vars['serviceid'] - Service ID
 * $vars['userid'] - Client ID
 * $vars['amount'] - Renewal amount
 */
function onServiceRenew($vars)
{
    $serviceId = $vars['serviceid'];
    
    // Extend service in external system
    your_module_extendService($serviceId);
    
    return true;
}
```

## Ticket Hooks

### TicketOpen

```php
/**
 * New support ticket opened
 * 
 * $vars['ticketid'] - Ticket ID
 * $vars['userid'] - Client ID
 * $vars['deptid'] - Department ID
 * $vars['subject'] - Ticket subject
 */
function onTicketOpen($vars)
{
    // Auto-assign based on keywords
    your_module_autoAssignTicket($vars['ticketid']);
    
    return true;
}
```

### TicketReply

```php
/**
 * Ticket reply added
 * 
 * $vars['ticketid'] - Ticket ID
 * $vars['userid'] - Client ID
 * $vars['message'] - Reply message
 */
function onTicketReply($vars)
{
    // Notify external system
    your_module_notifyTicketReply($vars['ticketid']);
    
    return true;
}
```

## Domain Hooks

### DomainRegister

```php
/**
 * Domain registration
 * 
 * $vars['domainid'] - Domain ID
 * $vars['domain'] - Domain name
 * $vars['userid'] - Client ID
 */
function onDomainRegister($vars)
{
    your_module_syncDomain($vars['domainid']);
    return true;
}
```

### DomainTransfer

```php
/**
 * Domain transfer
 * 
 * $vars['domainid'] - Domain ID
 * $vars['domain'] - Domain name
 * $vars['userid'] - Client ID
 */
function onDomainTransfer($vars)
{
    your_module_updateDomainTransfer($vars['domainid']);
    return true;
}
```

### DomainRenew

```php
/**
 * Domain renewal
 * 
 * $vars['domainid'] - Domain ID
 * $vars['domain'] - Domain name
 * $vars['years'] - Renewal years
 */
function onDomainRenew($vars)
{
    your_module_extendDomainRegistration($vars['domainid'], $vars['years']);
    return true;
}
```

## Admin Hooks

### AdminAreaHeader

```php
/**
 * Admin area header output
 * 
 * @return string HTML to prepend to header
 */
function onAdminAreaHeader()
{
    // Add CSS/JS assets
    $html = '<link rel="stylesheet" href="/modules/addons/your_module/assets/admin.css">';
    $html .= '<script src="/modules/addons/your_module/assets/admin.js"></script>';
    
    return $html;
}
```

### AdminAreaPage

```php
/**
 * Admin area page output
 * 
 * Add content to admin area pages
 */
function onAdminAreaPage($vars)
{
    $currentPage = $vars['filename'] ?? '';
    
    if ($currentPage === 'clients.php') {
        // Add column to clients list
        return '<script>
            $(document).ready(function() {
                $(".clientstable thead tr").append("<th>Your Module</th>");
            });
        </script>';
    }
    
    return '';
}
```

## Cron Hooks

### AfterCronJob

```php
/**
 * After cron execution
 * 
 * $vars['completed'] - Number of completed tasks
 * $vars['failed'] - Number of failed tasks
 */
function onAfterCronJob($vars)
{
    // Process queued module tasks
    your_module_processQueue();
    
    return true;
}
```

### HourlyCronJob

```php
/**
 * Hourly cron event
 */
function onHourlyCronJob($vars)
{
    // Process hourly tasks
    your_module_hourlyTask();
    
    return true;
}
```

### DailyCronJob

```php
/**
 * Daily cron event
 */
function onDailyCronJob($vars)
{
    // Process daily tasks (cleanup, reports, etc.)
    your_module_dailyTask();
    
    return true;
}
```

## Hook Parameters Reference

### Complete Client Variables

```php
$vars = [
    'userid'       => 1,
    'firstname'    => 'John',
    'lastname'     => 'Doe',
    'companyname'   => 'Example Inc',
    'email'        => 'john@example.com',
    'address1'     => '123 Main St',
    'address2'     => '',
    'city'         => 'New York',
    'state'        => 'NY',
    'country'      => 'US',
    'postcode'     => '10001',
    'phonenumber'  => '+1-555-1234',
    'mobilenumber' => '',
    'password'     => 'hashed',  // ClientAdd only
];
```

### Complete Invoice Variables

```php
$vars = [
    'invoiceid'     => 1,
    'invoicenum'     => 'INV-0001',
    'userid'         => 1,
    'status'         => 'Paid',
    'subtotal'       => 100.00,
    'tax'            => 10.00,
    'total'          => 110.00,
    'paymentmethod'   => 'stripe',
    'transid'        => 'txn_123456',
    'date'           => '2026-05-28',
    'duedate'        => '2026-06-07',
    'datepaid'       => '2026-05-28',
    'items'          => [],  // Array of line items
    'notes'          => '',
];
```

---

## Related Skills and Workflows

- `hooks-reference` - WHMCS hooks reference
- `cron-events-reference` - Cron event hooks
- `addon-module-developer-guide` - Addon development
- `module-hook-reference` - Module-specific hooks
