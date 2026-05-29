# WHMCS Notification Automation Workflow

## Overview
This workflow implements automated notification systems for WHMCS using multiple channels.

## Prerequisites
- WHMCS with email/SMS/notification capabilities
- External service integrations (optional)
- PHP development skills

## Step-by-Step Process

### Step 1: Create Notification Manager
```php
<?php
// /includes/notifications/NotificationManager.php

namespace WHMCS\Notifications;

class NotificationManager
{
    private $channels = [];
    private $queue = [];

    public function __construct()
    {
        $this->registerDefaultChannels();
    }

    private function registerDefaultChannels()
    {
        $this->channels = [
            'email' => new EmailChannel(),
            'sms' => new SMSChannel(),
            'push' => new PushChannel(),
            'webhook' => new WebhookChannel(),
            'slack' => new SlackChannel(),
            'discord' => new DiscordChannel()
        ];
    }

    /**
     * Send notification
     */
    public function send(string $channel, string $recipient, string $template, array $data = [], array $options = []): array
    {
        if (!isset($this->channels[$channel])) {
            throw new Exception("Unknown channel: {$channel}");
        }

        $result = $this->channels[$channel]->send($recipient, $template, $data, $options);

        // Log notification
        $this->logNotification($channel, $recipient, $template, $result);

        return $result;
    }

    /**
     * Send to multiple channels
     */
    public function sendMultiChannel(array $channels, string $recipient, string $template, array $data = []): array
    {
        $results = [];

        foreach ($channels as $channel) {
            $results[$channel] = $this->send($channel, $recipient, $template, $data);
        }

        return $results;
    }

    /**
     * Queue notification for later
     */
    public function queue(string $channel, string $recipient, string $template, array $data = [], $sendAt = null)
    {
        $this->queue[] = [
            'channel' => $channel,
            'recipient' => $recipient,
            'template' => $template,
            'data' => $data,
            'send_at' => $sendAt ?? date('Y-m-d H:i:s'),
            'created_at' => date('Y-m-d H:i:s')
        ];
    }

    /**
     * Process queued notifications
     */
    public function processQueue()
    {
        $pending = Capsule::table('mod_notification_queue')
            ->where('send_at', '<=', date('Y-m-d H:i:s'))
            ->where('sent', 0)
            ->get();

        foreach ($pending as $notification) {
            try {
                $this->send(
                    $notification->channel,
                    $notification->recipient,
                    $notification->template,
                    json_decode($notification->data, true)
                );

                Capsule::table('mod_notification_queue')
                    ->where('id', $notification->id)
                    ->update(['sent' => 1, 'sent_at' => date('Y-m-d H:i:s')]);
            } catch (Exception $e) {
                logActivity("Failed to send queued notification: " . $e->getMessage());
            }
        }
    }

    private function logNotification(string $channel, string $recipient, string $template, array $result)
    {
        Capsule::table('mod_notification_log')->insert([
            'channel' => $channel,
            'recipient' => $recipient,
            'template' => $template,
            'success' => $result['success'] ? 1 : 0,
            'error' => $result['error'] ?? null,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

### Step 2: Create Notification Channels
```php
<?php
// Email Channel
class EmailChannel
{
    public function send(string $recipient, string $template, array $data, array $options = []): array
    {
        try {
            sendEmail($recipient, $template, $data);

            return ['success' => true];
        } catch (Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }
}

// SMS Channel
class SMSChannel
{
    private $provider;

    public function __construct()
    {
        $this->provider = new TwilioProvider(); // or other SMS provider
    }

    public function send(string $recipient, string $template, array $data, array $options = []): array
    {
        try {
            // Get SMS template
            $message = $this->renderTemplate($template, $data);

            // Send SMS
            $result = $this->provider->send($recipient, $message);

            return ['success' => true, 'message_id' => $result['id']];
        } catch (Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    private function renderTemplate(string $template, array $data): string
    {
        $templates = [
            'service_suspended' => "Your service has been suspended. Please contact support.",
            'payment_reminder' => "Payment reminder: Invoice #{$data['invoice_id']} due on {$data['due_date']}.",
            'ticket_response' => "New response on ticket #{$data['ticket_id']}: {$data['message']}"
        ];

        return $templates[$template] ?? $template;
    }
}

// Webhook Channel
class WebhookChannel
{
    public function send(string $recipient, string $template, array $data, array $options = []): array
    {
        $ch = curl_init($recipient);

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode([
                'event' => $template,
                'data' => $data,
                'timestamp' => date('c')
            ]),
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return [
            'success' => $httpCode >= 200 && $httpCode < 300,
            'http_code' => $httpCode,
            'response' => $response
        ];
    }
}

// Slack Channel
class SlackChannel
{
    private $webhookUrl;

    public function __construct()
    {
        $this->webhookUrl = getConfig('slack_webhook_url');
    }

    public function send(string $recipient, string $template, array $data, array $options = []): array
    {
        $message = $this->formatSlackMessage($template, $data);

        $ch = curl_init($this->webhookUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode(['text' => $message]),
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return ['success' => true, 'response' => $response];
    }

    private function formatSlackMessage(string $template, array $data): string
    {
        $messages = [
            'new_order' => "New order received: {$data['order_id']} for {$data['amount']}",
            'payment_received' => "Payment received: {$data['amount']} from {$data['client_name']}",
            'service_suspended' => "Service suspended: {$data['service_name']} ({$data['reason']})",
            'ticket_created' => "New support ticket: #{$data['ticket_id']} - {$data['subject']}"
        ];

        return $messages[$template] ?? $template;
    }
}

// Discord Channel
class DiscordChannel
{
    private $webhookUrl;

    public function __construct()
    {
        $this->webhookUrl = getConfig('discord_webhook_url');
    }

    public function send(string $recipient, string $template, array $data, array $options = []): array
    {
        $embed = $this->createEmbed($template, $data);

        $ch = curl_init($this->webhookUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode(['embeds' => [$embed]]),
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return ['success' => true, 'response' => $response];
    }

    private function createEmbed(string $template, array $data): array
    {
        $colors = [
            'new_order' => 0x00ff00,
            'payment_received' => 0x0088ff,
            'service_suspended' => 0xff0000,
            'ticket_created' => 0xffaa00
        ];

        return [
            'title' => $data['title'] ?? 'WHMCS Notification',
            'description' => $data['message'] ?? '',
            'color' => $colors[$template] ?? 0x808080,
            'timestamp' => date('c'),
            'fields' => $this->createFields($data)
        ];
    }

    private function createFields(array $data): array
    {
        $fields = [];

        foreach ($data as $key => $value) {
            if (!in_array($key, ['title', 'message'])) {
                $fields[] = [
                    'name' => ucfirst(str_replace('_', ' ', $key)),
                    'value' => (string)$value,
                    'inline' => true
                ];
            }
        }

        return $fields;
    }
}
```

### Step 3: Create Notification Templates
```php
<?php
// /includes/hooks/notification_hooks.php

use WHMCS\Notifications\NotificationManager;

$notificationManager = new NotificationManager();

// New client notification
add_hook('ClientAdd', 1, function($vars) use ($notificationManager) {
    $client = getClientsDetails($vars['userid']);

    // Email notification
    $notificationManager->send('email', $vars['userid'], 'New Client Welcome', [
        'first_name' => $client['firstname'],
        'email' => $client['email']
    ]);

    // Admin Slack notification
    $notificationManager->send('slack', '', 'new_client', [
        'client_name' => $client['fullname'],
        'client_email' => $client['email'],
        'client_id' => $vars['userid']
    ]);
});

// Invoice paid notification
add_hook('InvoicePaid', 1, function($vars) use ($notificationManager) {
    $invoice = getInvoice($vars['invoiceid']);

    // Multi-channel notification
    $notificationManager->sendMultiChannel(
        ['email', 'sms', 'slack'],
        $vars['userid'],
        'Payment Received',
        [
            'invoice_id' => $vars['invoiceid'],
            'amount' => $invoice['total'],
            'payment_method' => $invoice['paymentmethod']
        ]
    );
});

// Service suspended notification
add_hook('ServiceSuspended', 1, function($vars) use ($notificationManager) {
    $notificationManager->send('email', $vars['userid'], 'Service Suspended', [
        'service_id' => $vars['serviceid'],
        'reason' => $vars['suspendreason'] ?? 'Payment overdue'
    ]);

    // Admin notification
    $notificationManager->send('discord', '', 'service_suspended', [
        'service_id' => $vars['serviceid'],
        'reason' => $vars['suspendreason'] ?? 'Unknown'
    ]);
});

// Support ticket notification
add_hook('TicketOpen', 1, function($vars) use ($notificationManager) {
    $notificationManager->send('slack', '', 'ticket_created', [
        'ticket_id' => $vars['ticketid'],
        'subject' => $vars['subject'],
        'priority' => $vars['priority']
    ]);
});
```

### Step 4: Create Scheduled Notifications
```php
<?php
// Scheduled notification hooks

add_hook('DailyCronJob', 1, function($vars) {
    $notificationManager = new NotificationManager();

    // Payment reminders
    $overdueInvoices = Capsule::table('tblinvoices')
        ->where('status', 'Overdue')
        ->where('duedate', '<', date('Y-m-d', strtotime('-7 days')))
        ->where('reminder_sent', '!=', date('Y-m-d'))
        ->get();

    foreach ($overdueInvoices as $invoice) {
        $daysOverdue = daysOverdue($invoice->duedate);

        $notificationManager->queue('email', $invoice->userid, 'Payment Reminder', [
            'invoice_id' => $invoice->id,
            'amount' => $invoice->total,
            'days_overdue' => $daysOverdue
        ], date('Y-m-d H:i:s', strtotime('+1 hour')));
    }

    // Domain renewal reminders
    $expiringDomains = Capsule::table('tbldomains')
        ->where('status', 'Active')
        ->where('expirydate', '<=', date('Y-m-d', strtotime('+30 days')))
        ->where('expirydate', '>=', date('Y-m-d'))
        ->where('renewal_notice_sent', 0)
        ->get();

    foreach ($expiringDomains as $domain) {
        $daysUntilExpiry = daysUntil($domain->expirydate);

        $notificationManager->queue('email', $domain->userid, 'Domain Renewal Reminder', [
            'domain' => $domain->domain,
            'expiry_date' => $domain->expirydate,
            'days_remaining' => $daysUntilExpiry
        ]);

        Capsule::table('tbldomains')
            ->where('id', $domain->id)
            ->update(['renewal_notice_sent' => 1]);
    }

    // Service expiry reminders
    $expiringServices = Capsule::table('tblhosting')
        ->where('domainstatus', 'Active')
        ->where('nextduedate', '<=', date('Y-m-d', strtotime('+7 days')))
        ->get();

    foreach ($expiringServices as $service) {
        $notificationManager->queue('email', $service->userid, 'Service Renewal Reminder', [
            'domain' => $service->domain,
            'next_due_date' => $service->nextduedate
        ]);
    }
});
```

### Step 5: Monitor Notification Delivery
```php
<?php
// Notification monitoring

function getNotificationStats(int $days = 7): array
{
    return Capsule::select("
        SELECT
            channel,
            COUNT(*) as total,
            SUM(success) as successful,
            SUM(NOT success) as failed
        FROM mod_notification_log
        WHERE created_at >= DATE_SUB(NOW(), INTERVAL ? DAY)
        GROUP BY channel
    ", [$days]);
}

add_hook('DailyCronJob', 1, function($vars) {
    $stats = getNotificationStats();

    // Alert on high failure rates
    foreach ($stats as $stat) {
        $failureRate = $stat->failed / $stat->total;

        if ($failureRate > 0.1) {
            sendAdminEmail('High Notification Failure Rate', [
                'channel' => $stat->channel,
                'failure_rate' => round($failureRate * 100, 1) . '%',
                'failed_count' => $stat->failed
            ]);
        }
    }
});
```

## Notification Best Practices

1. **Use appropriate channels** - Email for formal, SMS for urgent
2. **Don't over-notify** - Too many notifications cause fatigue
3. **Provide preferences** - Let users choose channels
4. **Include clear CTAs** - What should recipient do?
5. **Track delivery** - Monitor success/failure rates
6. **Handle failures** - Queue and retry failed notifications

## Related Workflows
- [WHMCS Email Automation](./whmcs-email-automation.md)
- [WHMCS SMS Automation](./whmcs-sms-automation.md)
- [WHMCS Slack/Discord/Telegram Automation](./whmcs-notification-automation.md)