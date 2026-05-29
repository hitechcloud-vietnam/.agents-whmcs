# WHMCS Advanced Hooks

Complete guide to advanced hook techniques.

## Overview

Master WHMCS hooks for powerful customizations.

## Hook Architecture

### Hook Manager

```php
<?php
/**
 * Advanced hook utilities
 */
class HookManager
{
    /**
     * Register hook with priority
     */
    public static function register(string $hookName, int $priority, callable $callback): void
    {
        add_hook($hookName, $priority, $callback);
    }
    
    /**
     * Register multiple hooks
     */
    public static function registerMany(array $hooks): void
    {
        foreach ($hooks as $hookName => $callbacks) {
            foreach ($callbacks as $priority => $callback) {
                add_hook($hookName, $priority, $callback);
            }
        }
    }
    
    /**
     * Conditional hook registration
     */
    public static function registerIf(bool $condition, string $hookName, int $priority, callable $callback): void
    {
        if ($condition) {
            add_hook($hookName, $priority, $callback);
        }
    }
}
```

## Hook Examples

### Client Lifecycle Hooks

```php
<?php
/**
 * Client registration hook
 */
add_hook('ClientAdd', 1, function($vars) {
    $clientId = $vars['userid'];
    
    // Send welcome email
    $client = Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();
    
    sendEmail('welcome', $client->email, [
        'client_name' => "{$client->firstname} {$client->lastname}",
    ]);
    
    // Create default settings
    Capsule::table('mod_client_settings')->insert([
        'client_id' => $clientId,
        'settings' => json_encode([
            'notifications' => true,
            'theme' => 'default',
        ]),
    ]);
    
    // Sync to CRM
    syncClientToCRM($clientId);
    
    logActivity("New client registered: {$client->email}");
});

/**
 * Client update hook
 */
add_hook('ClientEdit', 1, function($vars) {
    $clientId = $vars['userid'];
    
    // Detect changes
    $oldClient = Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();
    
    // Sync updates to CRM
    syncClientToCRM($clientId);
    
    // Log changes
    foreach ($vars as $key => $value) {
        if (isset($oldClient->$key) && $oldClient->$key !== $value) {
            logActivity("Client #{$clientId} updated: {$key} changed");
        }
    }
});

/**
 * Client deletion hook
 */
add_hook('ClientDelete', 1, function($vars) {
    $clientId = $vars['userid'];
    
    // Archive client data
    archiveClientData($clientId);
    
    // Remove from CRM
    removeClientFromCRM($clientId);
    
    logActivity("Client #{$clientId} deleted");
});
```

### Service Lifecycle Hooks

```php
<?php
/**
 * Service provisioning hook
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $params = $vars['params'];
    
    // Update service status
    Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->update(['domainstatus' => 'Active']);
    
    // Send notification
    $client = Capsule::table('tblclients')
        ->where('id', $params['userid'])
        ->first();
    
    sendEmail('service_activated', $client->email, [
        'domain' => $params['domain'],
        'service_url' => getServiceUrl($serviceId),
    ]);
    
    // Provision related services
    provisionAddons($serviceId);
    
    // Create welcome ticket
    createWelcomeTicket($serviceId);
    
    logActivity("Service provisioned: {$params['domain']}");
});

/**
 * Service suspension hook
 */
add_hook('AfterModuleSuspend', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();
    
    // Notify client
    $client = Capsule::table('tblclients')
        ->where('id', $service->userid)
        ->first();
    
    sendEmail('service_suspended', $client->email, [
        'domain' => $service->domain,
        'reason' => $vars['suspendreason'] ?? 'Payment overdue',
    ]);
    
    // Suspend related services
    suspendAddons($serviceId);
});

/**
 * Service termination hook
 */
add_hook('AfterModuleTerminate', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();
    
    // Create final backup
    createServiceBackup($serviceId);
    
    // Remove from monitoring
    removeFromMonitoring($service->domain);
    
    // Notify client
    $client = Capsule::table('tblclients')
        ->where('id', $service->userid)
        ->first();
    
    sendEmail('service_terminated', $client->email, [
        'domain' => $service->domain,
    ]);
    
    logActivity("Service terminated: {$service->domain}");
});
```

### Invoice Hooks

```php
<?php
/**
 * Invoice paid hook
 */
add_hook('InvoicePaid', 1, function($vars) {
    $invoiceId = $vars['invoice_id'];
    
    // Get invoice details
    $invoice = Capsule::table('tblinvoices')
        ->where('id', $invoiceId)
        ->first();
    
    // Activate pending services
    $items = Capsule::table('tblinvoiceitems')
        ->where('invoiceid', $invoiceId)
        ->where('type', 'Hosting')
        ->get();
    
    foreach ($items as $item) {
        Capsule::table('tblhosting')
            ->where('id', $item->relid)
            ->update(['domainstatus' => 'Active']);
    }
    
    // Send receipt
    sendEmail('invoice_receipt', $invoice->userid, [
        'invoice_id' => $invoiceId,
        'amount' => $invoice->total,
    ]);
    
    // Update affiliate commissions
    updateAffiliateCommission($invoice->userid, $invoice->total);
    
    // Sync to accounting
    syncToAccounting($invoiceId);
    
    logActivity("Invoice #{$invoiceId} paid");
});

/**
 * Invoice overdue hook
 */
add_hook('InvoiceOverdue', 1, function($vars) {
    $invoiceId = $vars['invoice_id'];
    $daysOverdue = $vars['days_overdue'];
    
    $invoice = Capsule::table('tblinvoices')
        ->where('id', $invoiceId)
        ->first();
    
    $client = Capsule::table('tblclients')
        ->where('id', $invoice->userid)
        ->first();
    
    // Send reminder based on days overdue
    if ($daysOverdue === 1) {
        sendEmail('payment_reminder_1', $client->email, [
            'invoice_id' => $invoiceId,
            'amount' => $invoice->total,
        ]);
    } elseif ($daysOverdue === 7) {
        sendEmail('payment_reminder_2', $client->email, [
            'invoice_id' => $invoiceId,
            'amount' => $invoice->total,
        ]);
        // Suspend services
        suspendOverdueServices($invoice->userid);
    } elseif ($daysOverdue === 14) {
        sendEmail('final_notice', $client->email, [
            'invoice_id' => $invoiceId,
            'amount' => $invoice->total,
        ]);
    }
});
```

### Authentication Hooks

```php
<?php
/**
 * Pre-authentication hook
 */
add_hook('UserAuthPreValidation', 1, function($vars) {
    $email = $vars['email'];
    $ip = $_SERVER['REMOTE_ADDR'];
    
    // Check for blocked IP
    $blocked = Capsule::table('mod_blocked_ips')
        ->where('ip_address', $ip)
        ->exists();
    
    if ($blocked) {
        logActivity("Blocked login attempt from {$ip}");
        return ['error' => 'Access denied'];
    }
    
    // Check for too many failed attempts
    $failures = Capsule::table('mod_login_attempts')
        ->where('email', $email)
        ->where('success', 0)
        ->where('created_at', '>', date('Y-m-d H:i:s', strtotime('-15 minutes')))
        ->count();
    
    if ($failures >= 5) {
        return ['error' => 'Too many failed attempts. Please try again later.'];
    }
    
    return null; // Continue with normal authentication
});

/**
 * Post-authentication hook
 */
add_hook('UserAuthSuccess', 1, function($vars) {
    $clientId = $vars['user_id'];
    
    // Log successful login
    Capsule::table('mod_login_attempts')->insert([
        'email' => $vars['email'],
        'success' => 1,
        'ip_address' => $_SERVER['REMOTE_ADDR'],
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    // Update last login
    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update(['lastlogin' => date('Y-m-d H:i:s')]);
    
    // Set session data
    $_SESSION['last_activity'] = time();
    $_SESSION['login_count'] = ($_SESSION['login_count'] ?? 0) + 1;
});
```

## Custom Hooks

### Define Custom Hook

```php
<?php
/**
 * Define custom hook
 */
function do_hook(string $hookName, array $params = []): void
{
    // Get all hooks for this event
    $hooks = Capsule::table('mod_custom_hooks')
        ->where('hook_name', $hookName)
        ->where('enabled', 1)
        ->get();
    
    foreach ($hooks as $hook) {
        $callback = unserialize($hook->callback);
        if (is_callable($callback)) {
            try {
                $callback($params);
            } catch (Exception $e) {
                logActivity("Custom hook error: {$hook->name} - " . $e->getMessage());
            }
        }
    }
}

/**
 * Trigger custom hook after service creation
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    do_hook('ServiceProvisioned', $vars);
});
```

## Hook Priority

```php
<?php
/**
 * Hook priority examples
 */

// Priority 1 - First execution
add_hook('ClientAdd', 1, function($vars) {
    // Validation first
});

// Priority 50 - Middle execution
add_hook('ClientAdd', 50, function($vars) {
    // Processing
});

// Priority 100 - Last execution
add_hook('ClientAdd', 100, function($vars) {
    // Final actions like logging
});
```

## Best Practices

1. **Use appropriate priority** - Order matters
2. **Handle exceptions** - Don't break WHMCS
3. **Log hook actions** - Track execution
4. **Return values** - Some hooks expect returns
5. **Check dependencies** - Ensure required data exists
6. **Test thoroughly** - Verify hook behavior

## Related Documentation

- [whmcs-advanced-hooks.md](whmcs-advanced-hooks.md)
- [whmcs-module-lifecycle.md](whmcs-module-lifecycle.md)
