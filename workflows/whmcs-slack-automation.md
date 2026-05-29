# WHMCS Slack Automation Workflow

## Overview
This workflow implements Slack integration for WHMCS notifications and team collaboration.

## Prerequisites
- WHMCS installation
- Slack workspace with webhook permissions
- Slack app configuration (optional for advanced features)

## Step-by-Step Process

### Step 1: Create Slack Manager
```php
<?php
// /includes/slack/SlackManager.php

namespace WHMCS\Slack;

class SlackManager
{
    private $webhookUrl;
    private $defaultChannel;

    public function __construct()
    {
        $this->webhookUrl = getConfig('slack_webhook_url');
        $this->defaultChannel = getConfig('slack_default_channel');
    }

    /**
     * Send message to Slack
     */
    public function send(string $message, array $options = []): array
    {
        $channel = $options['channel'] ?? $this->defaultChannel;

        $payload = [
            'text' => $message,
            'username' => $options['username'] ?? 'WHMCS Bot',
            'icon_emoji' => $options['icon'] ?? ':robot_face:'
        ];

        if ($channel) {
            $payload['channel'] = $channel;
        }

        return $this->sendPayload($payload);
    }

    /**
     * Send formatted message with blocks
     */
    public function sendFormatted(array $blocks, array $options = []): array
    {
        $payload = [
            'blocks' => $blocks,
            'username' => $options['username'] ?? 'WHMCS Bot',
            'icon_emoji' => $options['icon'] ?? ':robot_face:'
        ];

        $channel = $options['channel'] ?? $this->defaultChannel;
        if ($channel) {
            $payload['channel'] = $channel;
        }

        return $this->sendPayload($payload);
    }

    /**
     * Send invoice notification
     */
    public function invoiceNotification(int $invoiceId, string $event): array
    {
        $invoice = getInvoice($invoiceId);
        $client = getClientsDetails($invoice['userid']);

        $events = [
            'created' => [
                'icon' => ':memo:',
                'color' => '#439FE0',
                'title' => 'New Invoice Created'
            ],
            'paid' => [
                'icon' => ':moneybag:',
                'color' => '#36a64f',
                'title' => 'Invoice Paid'
            ],
            'overdue' => [
                'icon' => ':warning:',
                'color' => '#ff6b6b',
                'title' => 'Invoice Overdue'
            ]
        ];

        $config = $events[$event] ?? $events['created'];

        $blocks = [
            [
                'type' => 'header',
                'text' => [
                    'type' => 'plain_text',
                    'text' => $config['title'],
                    'emoji' => true
                ]
            ],
            [
                'type' => 'section',
                'fields' => [
                    ['type' => 'mrkdwn', 'text' => "*Invoice:*\n#{$invoiceId}"],
                    ['type' => 'mrkdwn', 'text' => "*Amount:*\n" . formatCurrency($invoice['total'])],
                    ['type' => 'mrkdwn', 'text' => "*Client:*\n{$client['fullname']}"],
                    ['type' => 'mrkdwn', 'text' => "*Status:*\n{$invoice['status']}"]
                ]
            ]
        ];

        return $this->sendFormatted($blocks, ['icon' => $config['icon']]);
    }

    /**
     * Send service notification
     */
    public function serviceNotification(int $serviceId, string $event): array
    {
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();

        $client = getClientsDetails($service->userid);

        $events = [
            'created' => ['icon' => ':white_check_mark:', 'title' => 'New Service Created'],
            'activated' => ['icon' => ':rocket:', 'title' => 'Service Activated'],
            'suspended' => ['icon' => ':pause_button:', 'title' => 'Service Suspended'],
            'terminated' => ['icon' => ':x:', 'title' => 'Service Terminated'],
            'upgraded' => ['icon' => ':arrow_up:', 'title' => 'Service Upgraded']
        ];

        $config = $events[$event] ?? ['icon' => ':bell:', 'title' => 'Service Update'];

        $blocks = [
            [
                'type' => 'header',
                'text' => [
                    'type' => 'plain_text',
                    'text' => $config['title'],
                    'emoji' => true
                ]
            ],
            [
                'type' => 'section',
                'fields' => [
                    ['type' => 'mrkdwn', 'text' => "*Service:*\n{$service->domain}"],
                    ['type' => 'mrkdwn', 'text' => "*Client:*\n{$client['fullname']}"],
                    ['type' => 'mrkdwn', 'text' => "*Status:*\n{$service->domainstatus}"],
                    ['type' => 'mrkdwn', 'text' => "*Amount:*\n" . formatCurrency($service->amount)]
                ]
            ]
        ];

        return $this->sendFormatted($blocks, ['icon' => $config['icon']]);
    }

    /**
     * Send support ticket notification
     */
    public function ticketNotification(int $ticketId, string $event): array
    {
        $ticket = Capsule::table('tbltickets')
            ->where('id', $ticketId)
            ->first();

        $events = [
            'created' => ['icon' => ':ticket:', 'title' => 'New Support Ticket'],
            'replied' => ['icon' => ':speech_balloon:', 'title' => 'Ticket Reply'],
            'closed' => ['icon' => ':lock:', 'title' => 'Ticket Closed']
        ];

        $config = $events[$event] ?? ['icon' => ':bell:', 'title' => 'Ticket Update'];

        $blocks = [
            [
                'type' => 'header',
                'text' => [
                    'type' => 'plain_text',
                    'text' => $config['title'],
                    'emoji' => true
                ]
            ],
            [
                'type' => 'section',
                'fields' => [
                    ['type' => 'mrkdwn', 'text' => "*Ticket:*\n#{$ticketId}"],
                    ['type' => 'mrkdwn', 'text' => "*Subject:*\n{$ticket->title}"],
                    ['type' => 'mrkdwn', 'text' => "*Priority:*\n{$ticket->urgency}"],
                    ['type' => 'mrkdwn', 'text' => "*Status:*\n{$ticket->status}"]
                ]
            ]
        ];

        return $this->sendFormatted($blocks, ['icon' => $config['icon']]);
    }

    private function sendPayload(array $payload): array
    {
        $ch = curl_init($this->webhookUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return ['success' => $response === 'ok'];
    }
}
```

### Step 2: Create Slack Hooks
```php
<?php
// /includes/hooks/slack_hooks.php

use WHMCS\Slack\SlackManager;

$slackManager = new SlackManager();

// Invoice events
add_hook('InvoiceCreated', 1, function($vars) use ($slackManager) {
    $slackManager->invoiceNotification($vars['invoiceid'], 'created');
});

add_hook('InvoicePaid', 1, function($vars) use ($slackManager) {
    $slackManager->invoiceNotification($vars['invoiceid'], 'paid');

    // Also send to #sales channel
    $slackManager->invoiceNotification($vars['invoiceid'], 'paid');
});

add_hook('InvoiceOverdue', 1, function($vars) use ($slackManager) {
    $slackManager->invoiceNotification($vars['invoiceid'], 'overdue');
});

// Service events
add_hook('AfterServiceCreate', 1, function($vars) use ($slackManager) {
    $slackManager->serviceNotification($vars['serviceid'], 'created');
});

add_hook('ServiceUnsuspended', 1, function($vars) use ($slackManager) {
    $slackManager->serviceNotification($vars['serviceid'], 'activated');
});

add_hook('ServiceSuspended', 1, function($vars) use ($slackManager) {
    $slackManager->serviceNotification($vars['serviceid'], 'suspended');
});

add_hook('ServiceTerminated', 1, function($vars) use ($slackManager) {
    $slackManager->serviceNotification($vars['serviceid'], 'terminated');
});

// Support tickets
add_hook('TicketOpen', 1, function($vars) use ($slackManager) {
    $slackManager->ticketNotification($vars['ticketid'], 'created');
});

add_hook('TicketReply', 1, function($vars) use ($slackManager) {
    $slackManager->ticketNotification($vars['ticketid'], 'replied');
});

add_hook('TicketClose', 1, function($vars) use ($slackManager) {
    $slackManager->ticketNotification($vars['ticketid'], 'closed');
});
```

### Step 3: Create Alert Channels
```php
<?php
// /includes/slack/SlackAlertChannels.php

class SlackAlertChannels
{
    private $slackManager;

    public function __construct()
    {
        $this->slackManager = new SlackManager();
    }

    /**
     * Alert for server issues
     */
    public function serverAlert(int $serverId, string $alertType, array $details): array
    {
        $server = Capsule::table('tblservers')
            ->where('id', $serverId)
            ->first();

        $blocks = [
            [
                'type' => 'header',
                'text' => [
                    'type' => 'plain_text',
                    'text' => ':warning: Server Alert',
                    'emoji' => true
                ]
            ],
            [
                'type' => 'section',
                'fields' => [
                    ['type' => 'mrkdwn', 'text' => "*Server:*\n{$server->name}"],
                    ['type' => 'mrkdwn', 'text' => "*Alert Type:*\n{$alertType}"],
                    ['type' => 'mrkdwn', 'text' => "*Details:*\n{$details['message']}"],
                    ['type' => 'mrkdwn', 'text' => "*Time:*\n" . date('Y-m-d H:i:s')]
                ]
            ]
        ];

        return $this->slackManager->sendFormatted($blocks, [
            'channel' => '#server-alerts',
            'icon' => ':warning:'
        ]);
    }

    /**
     * Alert for payment failures
     */
    public function paymentFailure(int $invoiceId, string $reason): array
    {
        $blocks = [
            [
                'type' => 'header',
                'text' => [
                    'type' => 'plain_text',
                    'text' => ':credit_card: Payment Failed',
                    'emoji' => true
                ]
            ],
            [
                'type' => 'section',
                'text' => [
                    'type' => 'mrkdwn',
                    'text' => "Invoice #{$invoiceId} payment failed: {$reason}"
                ]
            ]
        ];

        return $this->slackManager->sendFormatted($blocks, [
            'channel' => '#payments',
            'icon' => ':x:'
        ]);
    }

    /**
     * Daily summary report
     */
    public function dailySummary(): array
    {
        $today = date('Y-m-d');

        $stats = [
            'new_clients' => Capsule::table('tblclients')
                ->where('datecreated', $today)
                ->count(),
            'new_services' => Capsule::table('tblhosting')
                ->where('regdate', $today)
                ->count(),
            'revenue' => Capsule::table('tblinvoices')
                ->where('datepaid', $today)
                ->where('status', 'Paid')
                ->sum('total'),
            'support_tickets' => Capsule::table('tbltickets')
                ->where('date', $today)
                ->count()
        ];

        $blocks = [
            [
                'type' => 'header',
                'text' => [
                    'type' => 'plain_text',
                    'text' => ':chart_with_upwards_trend: Daily Summary - ' . date('M d, Y'),
                    'emoji' => true
                ]
            ],
            [
                'type' => 'section',
                'fields' => [
                    ['type' => 'mrkdwn', 'text' => "*New Clients:*\n{$stats['new_clients']}"],
                    ['type' => 'mrkdwn', 'text' => "*New Services:*\n{$stats['new_services']}"],
                    ['type' => 'mrkdwn', 'text' => "*Revenue:*\n$" . number_format($stats['revenue'], 2)],
                    ['type' => 'mrkdwn', 'text' => "*Tickets:*\n{$stats['support_tickets']}"]
                ]
            ]
        ];

        return $this->slackManager->sendFormatted($blocks, [
            'channel' => '#daily-reports',
            'icon' => ':bar_chart:'
        ]);
    }
}

// Add to cron for daily summary
add_hook('DailyCronJob', 1, function($vars) {
    $alerts = new SlackAlertChannels();
    $alerts->dailySummary();
});
```

## Slack Integration Best Practices

1. **Use channels appropriately** - Route alerts to correct channels
2. **Don't over-notify** - Avoid notification fatigue
3. **Include actionable info** - Add links to admin area
4. **Use rich formatting** - Blocks for better readability
5. **Set up thread replies** - Keep related items together
6. **Monitor bot activity** - Track message delivery

## Related Workflows
- [WHMCS Notification Automation](./whmcs-notification-automation.md)
- [WHMCS Discord Automation](./whmcs-discord-automation.md)
- [WHMCS Telegram Automation](./whmcs-telegram-automation.md)