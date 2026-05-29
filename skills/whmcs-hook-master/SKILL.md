# WHMCS Hook Master

## Overview
Master skill for WHMCS hook development. Covers hook system, hook types, hook parameters, best practices, and common patterns.

## Hook System Overview

WHMCS hooks allow you to execute custom code at specific points in the application lifecycle. Hooks are PHP files placed in the `/includes/hooks/` directory.

### Hook File Structure

```php
<?php
// /includes/hooks/your_custom_hook.php

/**
 * Hook Name: Custom Hook
 * Description: Description of what this hook does
 * Author: Your Name
 * Version: 1.0.0
 */

// Register the hook
add_hook('HookName', priority, function($params) {
    // Your hook logic here
    return $result;
});
```

## Common Hooks

### Client Hooks

```php
<?php
// /includes/hooks/client_hooks.php

/**
 * Before client registration
 */
add_hook('ClientAdd', 1, function(array $params) {
    // Validate client data
    $errors = [];

    // Check for spam email domains
    $emailDomain = explode('@', $params['email'])[1] ?? '';
    $blockedDomains = ['tempmail.com', 'throwaway.com'];
    if (in_array($emailDomain, $blockedDomains)) {
        $errors[] = 'Email domain not allowed';
    }

    // Validate phone number format
    if (!preg_match('/^\+?[1-9]\d{1,14}$/', $params['phonenumber'] ?? '')) {
        $errors[] = 'Invalid phone number format';
    }

    if (!empty($errors)) {
        return ['error' => implode(', ', $errors)];
    }

    // Add to mailing list
    addToMailingList($params['email'], $params['firstname'], $params['lastname']);

    return [];
});

/**
 * After client registration
 */
add_hook('ClientAreaFooterOutput', 1, function(array $params) {
    $client = \WHMCS\User\Client::find($params['userid']);

    if ($client) {
        // Track client activity
        logActivity("Client logged in: " . $client->email);
    }

    return '';
});

/**
 * Client login
 */
add_hook('ClientLogin', 1, function(array $params) {
    $client = \WHMCS\User\Client::find($params['userid']);

    // Update last login time
    $client->lastLogin = date('Y-m-d H:i:s');
    $client->loginCount = ($client->loginCount ?? 0) + 1;
    $client->save();

    // Send login notification for suspicious activity
    if ($client->loginCount > 10) {
        send_admin_notification('email', 'Suspicious login activity', "Client {$client->email} has logged in {$client->loginCount} times.");
    }

    return true;
});

/**
 * Client logout
 */
add_hook('ClientLogout', 1, function(array $params) {
    // Clean up session data
    unset($_SESSION['custom_data']);

    // Log the logout
    logActivity("Client logged out: User ID {$params['userid']}");

    return true;
});

/**
 * Client validation
 */
add_hook('ClientValidation', 1, function(array $params) {
    $errors = [];

    // Check minimum age (GDPR compliance)
    if (!empty($params['dateofbirth'])) {
        $birthDate = new \DateTime($params['dateofbirth']);
        $today = new \DateTime();
        $age = $birthDate->diff($today)->y;

        if ($age < 13) {
            $errors[] = 'Must be at least 13 years old';
        }
    }

    if (!empty($errors)) {
        return ['error' => implode(', ', $errors)];
    }

    return [];
});
```

### Order/Invoice Hooks

```php
<?php
// /includes/hooks/order_hooks.php

/**
 * Before order creation
 */
add_hook('PreOrderCreation', 1, function(array $params) {
    // Fraud detection
    $riskScore = 0;

    // Check for proxy/VPN (simplified example)
    if (isProxyIP($_SERVER['REMOTE_ADDR'])) {
        $riskScore += 30;
    }

    // Check for free email domains
    $emailDomain = explode('@', $params['email'])[1] ?? '';
    if (in_array($emailDomain, ['gmail.com', 'yahoo.com', 'hotmail.com'])) {
        $riskScore += 10;
    }

    // Check for high-risk country
    if (in_array($params['country'], ['XX', 'YY'])) {
        $riskScore += 20;
    }

    if ($riskScore >= 50) {
        return [
            'error' => 'Order flagged for manual review',
            'status' => 'Pending',
        ];
    }

    return [];
});

/**
 * After order creation
 */
add_hook('OrderCreation', 1, function(array $params) {
    $orderId = $params['orderid'];
    $clientId = $params['userid'];

    // Create welcome ticket
    create_support_ticket([
        'subject' => 'Welcome - Order #' . $orderId,
        'message' => 'Thank you for your order!',
        'client_id' => $clientId,
        'department_id' => 1,
    ]);

    // Assign to sales team
    assign_order_to_team($orderId, 'sales');

    // Log to CRM
    sync_to_crm($clientId, 'new_order', $orderId);

    return true;
});

/**
 * Order accepted (paid and provisioned)
 */
add_hook('AcceptOrder', 1, function(array $params) {
    $orderId = $params['orderid'];
    $order = \WHMCS\Order\Order::find($orderId);

    // Send order confirmation
    send_email('OrderConfirmation', $order->clientId, [
        'order_id' => $orderId,
    ]);

    // Update external systems
    sync_order_to_erp($order);

    // Add loyalty points
    add_loyalty_points($order->clientId, $order->total);

    return true;
});

/**
 * Invoice creation
 */
add_hook('InvoiceCreation', 1, function(array $params) {
    $invoiceId = $params['invoiceid'];
    $clientId = $params['userid'];

    // Add late fee for overdue invoices
    if ($params['duedate'] < date('Y-m-d')) {
        add_late_fee($invoiceId, 5.00);
    }

    return true;
});

/**
 * Invoice paid
 */
add_hook('InvoicePaid', 1, function(array $params) {
    $invoiceId = $params['invoiceid'];
    $invoice = \WHMCS\Billing\Invoice::find($invoiceId);

    // Update CRM
    sync_payment_to_crm($invoice->clientId, $params['amount']);

    // Award loyalty points
    add_loyalty_points($invoice->clientId, $params['amount'] * 0.01);

    // Send receipt
    send_email('PaymentReceipt', $invoice->clientId, [
        'invoice_id' => $invoiceId,
        'amount' => $params['amount'],
    ]);

    // Trigger affiliate commission
    process_affiliate_commission($invoice->clientId, $params['amount']);

    return true;
});

/**
 * Invoice overdue
 */
add_hook('InvoiceOverdue', 1, function(array $params) {
    $invoiceId = $params['invoiceid'];
    $daysOverdue = $params['daysOverdue'] ?? 0;

    // Send reminder emails at specific intervals
    $reminderDays = [1, 3, 7, 14, 30];

    if (in_array($daysOverdue, $reminderDays)) {
        send_email('InvoiceReminder', $params['userid'], [
            'invoice_id' => $invoiceId,
            'days_overdue' => $daysOverdue,
        ]);
    }

    // Suspend services after 14 days
    if ($daysOverdue >= 14) {
        $services = \WHMCS\Service\Service::where('clientId', $params['userid'])
            ->where('status', 'Active')
            ->get();

        foreach ($services as $service) {
            $service->suspend('Overdue Invoice');
        }
    }

    return true;
});

/**
 * Payment received
 */
add_hook('PaymentReceived', 1, function(array $params) {
    // Process payment through external systems
    process_payment_webhook($params);

    // Update accounting software
    sync_payment_to_accounting($params);

    return true;
});
```

### Service Hooks

```php
<?php
// /includes/hooks/service_hooks.php

/**
 * Before service creation
 */
add_hook('PreServiceCreate', 1, function(array $params) {
    // Check server capacity
    $server = \WHMCS\Server::find($params['server']);

    if (!$server || !$server->hasCapacity()) {
        return ['error' => 'No available server capacity'];
    }

    // Check domain availability
    if (!check_domain_availability($params['domain'])) {
        return ['error' => 'Domain not available'];
    }

    return [];
});

/**
 * After service creation
 */
add_hook('AfterServiceCreate', 1, function(array $params) {
    $serviceId = $params['serviceid'];

    // Create DNS records
    create_dns_records($params['domain'], $params['server']);

    // Setup monitoring
    setup_monitoring($serviceId);

    // Create welcome document
    create_service_welcome($serviceId);

    return true;
});

/**
 * Service provisioned
 */
add_hook('ServiceProvisioned', 1, function(array $params) {
    $serviceId = $params['serviceid'];
    $service = \WHMCS\Service\Service::find($serviceId);

    // Send welcome email with credentials
    send_email('ServiceWelcome', $service->clientId, [
        'service_id' => $serviceId,
        'domain' => $service->domain,
        'username' => $params['username'],
    ]);

    // Setup backup
    configure_backup($serviceId);

    return true;
});

/**
 * Service suspended
 */
add_hook('ServiceSuspended', 1, function(array $params) {
    $service = \WHMCS\Service\Service::find($params['serviceid']);

    // Notify client
    send_email('ServiceSuspended', $service->clientId, [
        'service_id' => $params['serviceid'],
        'reason' => $params['suspendreason'] ?? 'Overdue payment',
    ]);

    // Release resources
    release_resources($params['serviceid']);

    return true;
});

/**
 * Service unsuspended
 */
add_hook('ServiceUnsuspended', 1, function(array $params) {
    $service = \WHMCS\Service\Service::find($params['serviceid']);

    // Notify client
    send_email('ServiceUnsuspended', $service->clientId, [
        'service_id' => $params['serviceid'],
    ]);

    // Reallocate resources
    reallocate_resources($params['serviceid']);

    return true;
});

/**
 * Service terminated
 */
add_hook('ServiceTerminated', 1, function(array $params) {
    $serviceId = $params['serviceid'];

    // Backup data before termination
    backup_service_data($serviceId);

    // Notify client
    send_email('ServiceTerminated', $params['userid'], [
        'service_id' => $serviceId,
    ]);

    // Release all resources
    release_all_resources($serviceId);

    // Remove DNS records
    remove_dns_records($params['domain']);

    return true;
});

/**
 * Service renewed
 */
add_hook('ServiceRenewed', 1, function(array $params) {
    // Extend monitoring
    extend_monitoring($params['serviceid']);

    // Extend SSL certificates
    extend_ssl_certificates($params['serviceid']);

    return true;
});
```

### Ticket Hooks

```php
<?php
// /includes/hooks/ticket_hooks.php

/**
 * Ticket created
 */
add_hook('TicketOpen', 1, function(array $params) {
    $ticketId = $params['ticketid'];
    $departmentId = $params['deptid'];

    // Auto-assign based on keywords
    $keywords = [
        'billing' => 2,
        'technical' => 3,
        'sales' => 4,
    ];

    foreach ($keywords as $keyword => $assignDept) {
        if (stripos($params['subject'], $keyword) !== false) {
            reassign_ticket($ticketId, $assignDept);
            break;
        }
    }

    // Send acknowledgment
    send_email('TicketAcknowledgment', $params['userid'], [
        'ticket_id' => $ticketId,
    ]);

    // Log to analytics
    track_ticket_opened($ticketId);

    return true;
});

/**
 * Ticket replied
 */
add_hook('TicketAddReply', 1, function(array $params) {
    $ticketId = $params['ticketid'];

    // Update SLA
    update_ticket_sla($ticketId);

    // Notify client if assigned to staff
    if ($params['admin']) {
        send_email('TicketStaffReply', $params['userid'], [
            'ticket_id' => $ticketId,
        ]);
    }

    return true;
});

/**
 * Ticket closed
 */
add_hook('TicketClose', 1, function(array $params) {
    // Send satisfaction survey
    send_email('TicketSurvey', $params['userid'], [
        'ticket_id' => $params['ticketid'],
    ]);

    // Log resolution time
    log_resolution_time($params['ticketid']);

    return true;
});

/**
 * New ticket note
 */
add_hook('TicketAddNote', 1, function(array $params) {
    // Notify relevant staff
    notify_staff_ticket_note($params['ticketid']);

    return true;
});
```

### Custom Hooks

You can also create custom hooks that can be triggered from anywhere in your code:

```php
<?php
// /includes/hooks/custom_hooks.php

/**
 * Custom hook example
 * Can be triggered with: run_hook('CustomHookName', $params);
 */
add_hook('CustomHookName', 1, function(array $params) {
    // Your custom logic
    logActivity("Custom hook triggered: " . json_encode($params));

    return ['success' => true];
});
```

## Hook Priority

Hooks can have priorities from 1-100. Lower numbers execute first:

```php
// Execute first (priority 1)
add_hook('SomeHook', 1, function($params) {
    // Runs first
});

// Execute second (priority 50)
add_hook('SomeHook', 50, function($params) {
    // Runs second
});

// Execute last (priority 100)
add_hook('SomeHook', 100, function($params) {
    // Runs last
});
```

## Hook Return Values

Different hooks expect different return values:

- Some hooks return arrays to modify data or show errors
- Some hooks return strings to modify output
- Some hooks return boolean for success/failure
- Some hooks don't expect any return value

## Best Practices

1. **File Organization**: Group related hooks in separate files
2. **Error Handling**: Always wrap hook logic in try-catch
3. **Logging**: Log significant events for debugging
4. **Performance**: Keep hooks efficient; avoid heavy operations
5. **Idempotency**: Design hooks to be safely re-run
6. **Priority**: Use appropriate priorities for execution order
7. **Validation**: Validate all input parameters
8. **Security**: Check permissions before performing actions
9. **Documentation**: Comment your hook files
10. **Testing**: Test hooks in development first
