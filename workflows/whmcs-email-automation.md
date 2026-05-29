# WHMCS Email Automation Workflow

## Overview
This workflow implements comprehensive email automation for WHMCS including sequences, triggers, and templates.

## Prerequisites
- WHMCS with email configuration
- SMTP or email API credentials
- Email template creation access

## Step-by-Step Process

### Step 1: Create Email Automation Manager
```php
<?php
// /includes/email/EmailAutomationManager.php

namespace WHMCS\Email;

class EmailAutomationManager
{
    private $queue = [];

    /**
     * Queue welcome email sequence
     */
    public function queueWelcomeSequence(int $clientId): void
    {
        $sequence = [
            ['delay' => 0, 'template' => 'Welcome Email', 'vars' => []],
            ['delay' => 86400, 'template' => 'Getting Started Guide', 'vars' => []],
            ['delay' => 259200, 'template' => 'Feature Highlights', 'vars' => []],
            ['delay' => 604800, 'template' => 'Feedback Request', 'vars' => []]
        ];

        foreach ($sequence as $email) {
            $this->queueEmail($clientId, $email['template'], $email['vars'], $email['delay']);
        }
    }

    /**
     * Queue payment reminder sequence
     */
    public function queuePaymentReminders(int $clientId, int $invoiceId): void
    {
        $sequence = [
            ['days' => -3, 'template' => 'Invoice Reminder - 3 Days', 'priority' => 'normal'],
            ['days' => 0, 'template' => 'Invoice Due Today', 'priority' => 'high'],
            ['days' => 3, 'template' => 'Payment Overdue - 3 Days', 'priority' => 'high'],
            ['days' => 7, 'template' => 'Final Payment Notice', 'priority' => 'urgent'],
            ['days' => 14, 'template' => 'Service Suspension Warning', 'priority' => 'urgent']
        ];

        foreach ($sequence as $email) {
            $sendDate = date('Y-m-d H:i:s', strtotime("{$email['days']} days"));
            $this->queueEmail($clientId, $email['template'], ['invoice_id' => $invoiceId], $sendDate, $email['priority']);
        }
    }

    /**
     * Queue onboarding sequence
     */
    public function queueOnboardingSequence(int $clientId, int $serviceId): void
    {
        $sequence = [
            ['delay' => 0, 'template' => 'Service Activated', 'vars' => ['service_id' => $serviceId]],
            ['delay' => 3600, 'template' => 'Setup Instructions', 'vars' => ['service_id' => $serviceId]],
            ['delay' => 86400, 'template' => 'Day 1 Check-in', 'vars' => ['service_id' => $serviceId]],
            ['delay' => 604800, 'template' => '1 Week Review', 'vars' => ['service_id' => $serviceId]],
            ['delay' => 2592000, 'template' => '1 Month Check-in', 'vars' => ['service_id' => $serviceId]]
        ];

        foreach ($sequence as $email) {
            $this->queueEmail($clientId, $email['template'], $email['vars'], $email['delay']);
        }
    }

    /**
     * Queue single email
     */
    private function queueEmail(int $clientId, string $template, array $vars, $delay, string $priority = 'normal')
    {
        if (is_numeric($delay)) {
            $sendAt = date('Y-m-d H:i:s', time() + $delay);
        } else {
            $sendAt = $delay;
        }

        Capsule::table('mod_email_queue')->insert([
            'client_id' => $clientId,
            'template' => $template,
            'vars' => json_encode($vars),
            'priority' => $priority,
            'send_at' => $sendAt,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    /**
     * Cancel queued emails
     */
    public function cancelQueuedEmails(int $clientId, string $template = null): int
    {
        $query = Capsule::table('mod_email_queue')
            ->where('client_id', $clientId)
            ->where('sent', 0);

        if ($template) {
            $query->where('template', $template);
        }

        return $query->update(['cancelled' => 1]);
    }
}
```

### Step 2: Create Email Hooks
```php
<?php
// /includes/hooks/email_automation_hooks.php

use WHMCS\Email\EmailAutomationManager;

$emailManager = new EmailAutomationManager();

// New client - start welcome sequence
add_hook('ClientAdd', 1, function($vars) use ($emailManager) {
    $emailManager->queueWelcomeSequence($vars['userid']);
});

// Service activated - start onboarding
add_hook('AfterServiceCreate', 1, function($vars) use ($emailManager) {
    $service = Capsule::table('tblhosting')
        ->where('id', $vars['serviceid'])
        ->first();

    if ($service->domainstatus === 'Active') {
        $emailManager->queueOnboardingSequence($vars['userid'], $vars['serviceid']);
    }
});

// Invoice created - queue reminders
add_hook('InvoiceCreated', 1, function($vars) use ($emailManager) {
    $email = Capsule::table('tblinvoices')
        ->where('id', $vars['invoiceid'])
        ->first();

    if ($email->status === 'Unpaid') {
        $emailManager->queuePaymentReminders($email->userid, $vars['invoiceid']);
    }
});

// Payment received - cancel pending reminders
add_hook('InvoicePaid', 1, function($vars) use ($emailManager) {
    $invoice = Capsule::table('tblinvoices')
        ->where('id', $vars['invoiceid'])
        ->first();

    // Cancel payment reminder emails
    $emailManager->cancelQueuedEmails($invoice->userid, 'Invoice Reminder%');
    $emailManager->cancelQueuedEmails($invoice->userid, 'Payment Overdue%');
});

// Renewal reminder sequence
add_hook('DailyCronJob', 1, function($vars) use ($emailManager) {
    // Find services due in 30 days
    $services = Capsule::table('tblhosting')
        ->where('domainstatus', 'Active')
        ->where('nextduedate', date('Y-m-d', strtotime('+30 days')))
        ->whereNotExists(function($q) {
            $q->select(Capsule::raw(1))
                ->from('mod_email_queue')
                ->whereRaw('JSON_EXTRACT(vars, "$.service_id") = tblhosting.id')
                ->where('template', 'Renewal Reminder - 30 Days')
                ->where('sent', 1);
        })
        ->get();

    foreach ($services as $service) {
        $emailManager->queueEmail(
            $service->userid,
            'Renewal Reminder - 30 Days',
            ['service_id' => $service->id, 'domain' => $service->domain],
            0 // Send immediately
        );
    }
});
```

### Step 3: Create Email Sequence Manager
```php
<?php
// /includes/email/EmailSequenceManager.php

class EmailSequenceManager
{
    private $sequences = [];

    public function __construct()
    {
        $this->loadSequences();
    }

    private function loadSequences()
    {
        $this->sequences = [
            'welcome' => [
                'name' => 'Welcome Sequence',
                'emails' => [
                    ['template' => 'Welcome', 'delay' => 0],
                    ['template' => 'Getting Started', 'delay' => 3600],
                    ['template' => 'Top Features', 'delay' => 86400],
                    ['template' => 'Support Info', 'delay' => 172800]
                ]
            ],
            'upsell' => [
                'name' => 'Upsell Sequence',
                'trigger' => 'service_upgrade_offer',
                'emails' => [
                    ['template' => 'Upgrade Offer', 'delay' => 0],
                    ['template' => 'Upgrade Benefits', 'delay' => 86400],
                    ['template' => 'Limited Time Offer', 'delay' => 172800]
                ]
            ],
            'reengagement' => [
                'name' => 'Re-engagement Sequence',
                'trigger' => 'inactive_90_days',
                'emails' => [
                    ['template' => 'We Miss You', 'delay' => 0],
                    ['template' => 'New Features', 'delay' => 259200],
                    ['template' => 'Special Offer', 'delay' => 432000],
                    ['template' => 'Final Notice', 'delay' => 604800]
                ]
            ]
        ];
    }

    /**
     * Start a sequence for a client
     */
    public function startSequence(int $clientId, string $sequenceName, array $context = []): array
    {
        if (!isset($this->sequences[$sequenceName])) {
            return ['success' => false, 'error' => 'Unknown sequence'];
        }

        $sequence = $this->sequences[$sequenceName];
        $queueIds = [];

        foreach ($sequence['emails'] as $index => $email) {
            $sendAt = date('Y-m-d H:i:s', time() + $email['delay']);

            $queueId = Capsule::table('mod_email_sequences')->insertGetId([
                'client_id' => $clientId,
                'sequence_name' => $sequenceName,
                'step' => $index,
                'template' => $email['template'],
                'send_at' => $sendAt,
                'status' => 'queued',
                'context' => json_encode($context),
                'created_at' => date('Y-m-d H:i:s')
            ]);

            $queueIds[] = $queueId;
        }

        return ['success' => true, 'queue_ids' => $queueIds];
    }

    /**
     * Stop a sequence for a client
     */
    public function stopSequence(int $clientId, string $sequenceName): int
    {
        return Capsule::table('mod_email_sequences')
            ->where('client_id', $clientId)
            ->where('sequence_name', $sequenceName)
            ->where('status', 'queued')
            ->update(['status' => 'stopped']);
    }
}
```

### Step 4: Process Email Queue (Cron)
```php
<?php
// /includes/email/process_email_queue.php

add_hook('CronJobHourly', 1, function($vars) {
    // Process main email queue
    $pendingEmails = Capsule::table('mod_email_queue')
        ->where('send_at', '<=', date('Y-m-d H:i:s'))
        ->where('sent', 0)
        ->where('cancelled', 0)
        ->orderBy('priority', 'DESC')
        ->orderBy('send_at')
        ->limit(100)
        ->get();

    foreach ($pendingEmails as $email) {
        try {
            $vars = json_decode($email->vars, true);

            sendEmail($email->client_id, $email->template, $vars);

            Capsule::table('mod_email_queue')
                ->where('id', $email->id)
                ->update(['sent' => 1, 'sent_at' => date('Y-m-d H:i:s')]);

            logActivity("Email sent: {$email->template} to client {$email->client_id}");
        } catch (Exception $e) {
            Capsule::table('mod_email_queue')
                ->where('id', $email->id)
                ->update(['error' => $e->getMessage(), 'attempts' => $email->attempts + 1]);

            if ($email->attempts >= 3) {
                // Mark as failed after 3 attempts
                Capsule::table('mod_email_queue')
                    ->where('id', $email->id)
                    ->update(['sent' => -1]);
            }
        }
    }

    // Process email sequences
    processEmailSequences();

    return "Processed {$pendingEmails->count()} emails";
});

function processEmailSequences()
{
    $pending = Capsule::table('mod_email_sequences')
        ->where('send_at', '<=', date('Y-m-d H:i:s'))
        ->where('status', 'queued')
        ->limit(50)
        ->get();

    foreach ($pending as $item) {
        try {
            $context = json_decode($item->context, true);

            sendEmail($item->client_id, $item->template, $context);

            Capsule::table('mod_email_sequences')
                ->where('id', $item->id)
                ->update(['status' => 'sent', 'sent_at' => date('Y-m-d H:i:s')]);
        } catch (Exception $e) {
            logActivity("Sequence email failed: " . $e->getMessage());
        }
    }
}
```

### Step 5: Create Email Personalization
```php
<?php
// Email personalization helpers

function personalizeEmail(string $template, array $clientData, array $additionalData = []): array
{
    $personalizations = [
        'first_name' => $clientData['firstname'] ?? '',
        'full_name' => ($clientData['firstname'] ?? '') . ' ' . ($clientData['lastname'] ?? ''),
        'email' => $clientData['email'] ?? '',
        'company' => $clientData['companyname'] ?? '',
        'client_id' => $clientData['id'] ?? '',
    ];

    return array_merge($personalizations, $additionalData);
}

// Usage
add_hook('InvoiceCreated', 1, function($vars) {
    $invoice = getInvoice($vars['invoiceid']);
    $client = getClientsDetails($invoice['userid']);

    $personalizedVars = personalizeEmail('Invoice Created', $client, [
        'invoice_id' => $vars['invoiceid'],
        'amount' => $invoice['total'],
        'due_date' => $invoice['duedate']
    ]);

    sendEmail($invoice['userid'], 'Invoice Created', $personalizedVars);
});
```

## Email Automation Best Practices

1. **Segment your lists** - Send relevant content to relevant groups
2. **Set up double opt-in** - Ensure email validity
3. **Monitor deliverability** - Track bounce and complaint rates
4. **Personalize content** - Use merge fields for relevance
5. **Clean your list** - Remove inactive subscribers
6. **Test before sending** - Preview and test emails

## Related Workflows
- [WHMCS Notification Automation](./whmcs-notification-automation.md)
- [WHMCS SMS Automation](./whmcs-sms-automation.md)
- [WHMCS Push Automation](./whmcs-push-automation.md)