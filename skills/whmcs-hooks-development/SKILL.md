# WHMCS Hooks Development Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for creating and managing WHMCS hooks for automation and integration.

## When to Use

- Adding custom automation logic
- Integrating with external services on events
- Modifying WHMCS behavior
- Scheduling periodic tasks

## Hook File Structure

```php
<?php
// hooks.php
if (!defined("WHMCS")) { die("Direct access denied"); }

add_hook('{HookName}', {priority}, function($vars) {
    // Hook logic here
    // $vars contains hook-specific parameters
});
```

## Common Hooks Reference

### Client Hooks

```php
// Client Registration
add_hook('ClientAdd', 1, function($vars) {
    $userId = $vars['userid'];
    $email = $vars['email'];
    $firstName = $vars['firstname'];
    $lastName = $vars['lastname'];

    // Send welcome email
    // Create account in external system
    // Assign default package
});

// Client Update
add_hook('ClientEdit', 1, function($vars) {
    $userId = $vars['userid'];
    // Sync changes to external system
});

// Client Login
add_hook('ClientLogin', 1, function($vars) {
    $userId = $vars['userid'];
    // Log login activity
    // Update last login timestamp
});

// Client Deletion
add_hook('ClientDelete', 1, function($vars) {
    $userId = $vars['userid'];
    // Clean up external accounts
});
```

### Service Hooks

```php
// After Service Create
add_hook('AfterModuleCreate', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $userId = $vars['userid'];
    $params = $vars['params'];

    // Provision service
    // Send welcome email
    // Create billing record
});

// After Service Suspend
add_hook('AfterModuleSuspend', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    // Notify customer
    // Update monitoring
});

// After Service Unsuspend
add_hook('AfterModuleUnsuspend', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    // Reactivate service
    // Send reactivation email
});

// After Service Terminate
add_hook('AfterModuleTerminate', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    // Clean up resources
    // Archive data
});

// After Password Change
add_hook('AfterModuleChangePassword', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $password = $vars['password'];
    // Update service password
});
```

### Invoice Hooks

```php
// Invoice Created
add_hook('InvoiceCreation', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];
    $userId = $vars['userid'];
    // Add custom line items
    // Apply discounts
});

// Invoice Paid
add_hook('InvoicePaid', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];
    $userId = $vars['userid'];
    $amount = $vars['amount'];

    // Fulfill order
    // Send receipt
    // Sync to accounting
});

// Add Transaction
add_hook('AddTransaction', 1, function($vars) {
    $transId = $vars['transid'];
    // Log transaction
    // Update balance
});
```

### Ticket Hooks

```php
// New Ticket
add_hook('TicketOpen', 1, function($vars) {
    $ticketId = $vars['ticketid'];
    $userId = $vars['userid'];
    $deptId = $vars['deptid'];

    // Auto-assign
    // Send notification
    // Create internal ticket
});

// Ticket Reply
add_hook('TicketReply', 1, function($vars) {
    $ticketId = $vars['ticketid'];
    // Notify staff
    // Log response time
});
```

### Order Hooks

```php
// New Order
add_hook('NewOrder', 1, function($vars) {
    $orderId = $vars['orderid'];
    $userId = $vars['userid'];
    // Validate order
    // Check fraud
});

// Accept Order
add_hook('AcceptOrder', 1, function($vars) {
    $orderId = $vars['orderid'];
    // Fulfill order
});
```

### Domain Hooks

```php
// Domain Registration
add_hook('DomainRegistration', 1, function($vars) {
    $domainId = $vars['domainid'];
    // Setup DNS
    // Configure email
});

// Domain Transfer
add_hook('DomainTransferAccept', 1, function($vars) {
    $domainId = $vars['domainid'];
    // Complete transfer
});

// Domain Renewal
add_hook('DomainRenewal', 1, function($vars) {
    $domainId = $vars['domainid'];
    // Renew with registrar
});
```

### Cron Hooks

```php
// Daily Cron
add_hook('DailyCronJob', 1, function($vars) {
    // Generate reports
    // Process queue
    // Sync data
});

// Hourly Cron
add_hook('HourlyCronJob', 1, function($vars) {
    // Quick tasks
    // Cleanup
});
```

## Custom Hook Registration

```php
// Fire custom hook
use WHMCS\Module\Hook;

function fireCustomHook(string $hookName, array $data): void {
    // Log the hook
    Capsule::table('mod_module_hooks')->insert([
        'hook_name' => $hookName,
        'hook_data' => json_encode($data),
        'fired_at' => date('Y-m-d H:i:s'),
    ]);

    // Trigger WHMCS hooks
    run_hook('CustomHook_' . $hookName, $data);
}

// Register custom hook handler
add_hook('CustomHook_process_data', 1, function($vars) {
    // Handle custom hook
});
```

## Hook Priority

| Priority | Use Case |
|----------|----------|
| 0-1 | Critical operations, early exit |
| 5-10 | Standard processing |
| 50-100 | Late processing, logging |
| 1000+ | Debugging, monitoring |

## Checklist

- [ ] hooks.php file created
- [ ] check_token() for user-facing hooks
- [ ] Error handling with try/catch
- [ ] Logging for debugging
- [ ] Priority appropriate for operation
- [ ] Return values when expected

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-automation
- whmcs-cron-setup