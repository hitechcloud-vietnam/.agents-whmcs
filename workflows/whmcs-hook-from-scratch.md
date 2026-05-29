# WHMCS Hook Module From Scratch Workflow

## Description
Create a comprehensive hook module for WHMCS to customize functionality.

## Prerequisites
- WHMCS 7.0+
- PHP 8.1+
- Understanding of WHMCS hooks system

## Steps

### Step 1: Create Hook File
```php
<?php
/**
 * WHMCS Custom Hooks Module
 * Add this file to: /var/www/whmcs/includes/hooks/
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Hook: After client registration
 */
add_hook('ClientAreaRegistrationCompleted', 1, function($vars) {
    $clientId = $vars['userid'];
    $email = $vars['email'];
    
    // Log registration
    logActivity("New client registered: $email (ID: $clientId)");
    
    // Send welcome email
    sendEmail('Welcome Email', $clientId);
    
    // Create custom record
    Capsule::table('mod_custom_registrations')->insert([
        'client_id' => $clientId,
        'registered_at' => date('Y-m-d H:i:s'),
        'source' => $_COOKIE['affiliate_referral'] ?? 'direct',
    ]);
});

/**
 * Hook: Before client login
 */
add_hook('PreLogin', 1, function($vars) {
    $email = $vars['email'];
    
    // Check for locked account
    $locked = Capsule::table('mod_locked_accounts')
        ->where('email', $email)
        ->first();
    
    if ($locked && $locked->locked_until > date('Y-m-d H:i:s')) {
        return [
            'error' => 'Account is temporarily locked. Please try again later.',
        ];
    }
    
    // Track login attempt
    Capsule::table('mod_login_attempts')->insert([
        'email' => $email,
        'ip' => $_SERVER['REMOTE_ADDR'],
        'timestamp' => date('Y-m-d H:i:s'),
    ]);
});

/**
 * Hook: After successful login
 */
add_hook('AfterLogin', 1, function($vars) {
    $userId = $vars['userid'];
    
    // Update last login
    Capsule::table('tblclients')
        ->where('id', $userId)
        ->update(['lastlogin' => date('Y-m-d H:i:s')]);
    
    // Clear failed attempts
    Capsule::table('mod_login_attempts')
        ->where('email', $vars['email'])
        ->delete();
});

/**
 * Hook: After order placement
 */
add_hook('OrderPlaced', 1, function($vars) {
    $orderId = $vars['orderId'];
    
    // Send order confirmation to team
    $message = "New order #$orderId received";
    sendNotificationSlack($message);
    
    // Create tracking record
    Capsule::table('mod_order_tracking')->insert([
        'order_id' => $orderId,
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
});

/**
 * Hook: After order is paid
 */
add_hook('OrderPaid', 1, function($vars) {
    $orderId = $vars['orderId'];
    
    // Update tracking
    Capsule::table('mod_order_tracking')
        ->where('order_id', $orderId)
        ->update(['status' => 'paid', 'paid_at' => date('Y-m-d H:i:s')]);
    
    // Trigger provisioning
    logActivity("Order #$orderId paid - provisioning triggered");
});

/**
 * Hook: After service provisioning
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $module = $vars['moduletype'];
    
    // Send welcome email with service details
    $service = Capsule::table('tblhosting')->find($serviceId);
    sendEmail('Service Welcome', $service->userid, [
        'service_id' => $serviceId,
        'module' => $module,
    ]);
});

/**
 * Hook: Before invoice generation
 */
add_hook('PreInvoiceCreation', 1, function($vars) {
    // Add custom line items
    $items = [];
    
    // Add loyalty discount for long-term clients
    $clientId = $vars['userid'];
    $client = Capsule::table('tblclients')->find($clientId);
    
    $yearsSinceCreation = (time() - strtotime($client->createdat)) / (365 * 24 * 60 * 60);
    
    if ($yearsSinceCreation > 2) {
        $items[] = [
            'description' => 'Loyalty Discount (2+ years)',
            'amount' => -5.00, // $5 discount
        ];
    }
    
    if (!empty($items)) {
        return ['items' => $items];
    }
});

/**
 * Hook: After invoice creation
 */
add_hook('InvoiceCreated', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];
    
    // Log invoice creation
    logActivity("Invoice #$invoiceId created");
    
    // Send reminder if high value
    $invoice = localAPI('GetInvoice', ['invoiceid' => $invoiceId]);
    
    if ($invoice['total'] > 1000) {
        sendEmail('High Value Invoice Alert', null, [
            'invoice_id' => $invoiceId,
            'amount' => $invoice['total'],
        ]);
    }
});

/**
 * Hook: Before ticket creation
 */
add_hook('TicketOpenPreSubmit', 1, function($vars) {
    // Check for spam
    $subject = $vars['subject'];
    $message = $vars['message'];
    
    if (clicodes_is_spam($subject, $message)) {
        return [
            'error' => 'Your message appears to be spam. Please try again.',
        ];
    }
});

/**
 * Hook: After ticket creation
 */
add_hook('TicketOpen', 1, function($vars) {
    $ticketId = $vars['ticketid'];
    
    // Auto-assign based on keywords
    $keywords = [
        'billing' => 'billing_dept',
        'technical' => 'tech_support',
        'sales' => 'sales_team',
    ];
    
    foreach ($keywords as $keyword => $dept) {
        if (stripos($vars['subject'], $keyword) !== false) {
            Capsule::table('tbltickets')
                ->where('id', $ticketId)
                ->update(['did' => getDepartmentId($dept)]);
            break;
        }
    }
    
    // Notify Slack channel
    sendNotificationSlack("New ticket: #{$ticketId} - {$vars['subject']}");
});

/**
 * Hook: Before email sending
 */
add_hook('EmailPreSend', 1, function($vars) {
    // Add email tracking pixel
    if ($vars['type'] === 'support') {
        $vars['body'] .= '<img src="https://example.com/track/' . md5($vars['relid']) . '.gif" width="1" height="1">';
    }
    
    // Translate emails based on client language
    if (isset($vars['clientid'])) {
        $client = Capsule::table('tblclients')->find($vars['clientid']);
        if ($client && $client->language !== 'english') {
            $vars['subject'] = translateEmail($vars['subject'], $client->language);
            $vars['body'] = translateEmail($vars['body'], $client->language);
        }
    }
});

/**
 * Hook: Cron job execution
 */
add_hook('AfterCronJob', 1, function($vars) {
    // Cleanup old records
    Capsule::table('mod_login_attempts')
        ->where('timestamp', '<', date('Y-m-d H:i:s', strtotime('-24 hours')))
        ->delete();
    
    // Unlock old locked accounts
    Capsule::table('mod_locked_accounts')
        ->where('locked_until', '<', date('Y-m-d H:i:s'))
        ->delete();
});

/**
 * Helper function: Check for spam
 */
function clicodes_is_spam($subject, $message) {
    $spamKeywords = ['casino', 'viagra', 'lottery'];
    
    foreach ($spamKeywords as $keyword) {
        if (stripos($subject, $keyword) !== false || stripos($message, $keyword) !== false) {
            return true;
        }
    }
    
    return false;
}

/**
 * Helper function: Send Slack notification
 */
function sendNotificationSlack($message) {
    $webhookUrl = Capsule::table('tbladdonmodules')
        ->where('module', 'slack_integration')
        ->where('setting', 'webhook_url')
        ->first();
    
    if (!$webhookUrl) {
        return;
    }
    
    $ch = curl_init($webhookUrl->value);
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode(['text' => $message]),
        CURLOPT_RETURNTRANSFER => true,
    ]);
    curl_exec($ch);
    curl_close($ch);
}
```

### Step 2: Install Hooks
```bash
# Method 1: Copy to hooks directory
cp clicodes_hooks.php /var/www/whmcs/includes/hooks/clicodes_hooks.php

# Method 2: Create as addon module hooks
# Add hooks via modules/addons/yourmodule/hooks.php
```

### Step 3: Common Hook Points

| Hook | Timing | Use Case |
|------|--------|----------|
| PreLogin | Before login | Validation, rate limiting |
| AfterLogin | After login | Session, tracking |
| ClientAreaRegistrationCompleted | After registration | Welcome, onboarding |
| OrderPlaced | After order | Notification, tracking |
| OrderPaid | After payment | Provisioning, alerts |
| AfterModuleCreate | After provisioning | Welcome emails |
| InvoiceCreated | After invoice | Notifications |
| TicketOpen | After ticket | Routing, notifications |
| EmailPreSend | Before email | Modification, tracking |
| AfterCronJob | After cron | Cleanup, maintenance |

## Tags
- hooks
- customization
- automation
- development