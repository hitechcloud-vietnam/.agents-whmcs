# WHMCS Discord Automation Workflow

## Overview
This workflow implements Discord webhook integration for WHMCS notifications.

## Prerequisites
- WHMCS installation
- Discord server with webhook permissions
- Discord webhook URL

## Step-by-Step Process

### Step 1: Create Discord Manager
```php
<?php
// /includes/discord/DiscordManager.php

namespace WHMCS\Discord;

class DiscordManager
{
    private $webhooks = [];

    public function __construct()
    {
        $this->loadWebhooks();
    }

    private function loadWebhooks()
    {
        $this->webhooks = [
            'default' => getConfig('discord_webhook_url'),
            'alerts' => getConfig('discord_alerts_webhook'),
            'sales' => getConfig('discord_sales_webhook'),
            'support' => getConfig('discord_support_webhook')
        ];
    }

    /**
     * Send message to Discord
     */
    public function send(string $message, string $channel = 'default', array $options = []): array
    {
        $webhookUrl = $this->webhooks[$channel] ?? $this->webhooks['default'];

        if (!$webhookUrl) {
            return ['success' => false, 'error' => 'Webhook not configured'];
        }

        $payload = [
            'content' => $message,
            'username' => $options['username'] ?? 'WHMCS',
            'avatar_url' => $options['avatar'] ?? null
        ];

        return $this->sendPayload($webhookUrl, $payload);
    }

    /**
     * Send rich embed message
     */
    public function sendEmbed(array $embed, string $channel = 'default'): array
    {
        $webhookUrl = $this->webhooks[$channel] ?? $this->webhooks['default'];

        if (!$webhookUrl) {
            return ['success' => false, 'error' => 'Webhook not configured'];
        }

        $payload = [
            'username' => 'WHMCS',
            'embeds' => [$embed]
        ];

        return $this->sendPayload($webhookUrl, $payload);
    }

    /**
     * Send invoice notification
     */
    public function invoiceNotification(int $invoiceId, string $event): array
    {
        $invoice = getInvoice($invoiceId);
        $client = getClientsDetails($invoice['userid']);

        $events = [
            'created' => ['color' => 0x3498db, 'title' => 'Invoice Created', 'icon' => ':memo:'],
            'paid' => ['color' => 0x27ae60, 'title' => 'Payment Received', 'icon' => ':moneybag:'],
            'overdue' => ['color' => 0xe74c3c, 'title' => 'Invoice Overdue', 'icon' => ':warning:'],
            'cancelled' => ['color' => 0x95a5a6, 'title' => 'Invoice Cancelled', 'icon' => ':no_entry:']
        ];

        $config = $events[$event] ?? $events['created'];

        $embed = [
            'title' => "{$config['icon']} {$config['title']}",
            'color' => $config['color'],
            'fields' => [
                ['name' => 'Invoice', 'value' => "#{$invoiceId}", 'inline' => true],
                ['name' => 'Amount', 'value' => formatCurrency($invoice['total']), 'inline' => true],
                ['name' => 'Client', 'value' => $client['fullname'], 'inline' => true],
                ['name' => 'Status', 'value' => ucfirst($invoice['status']), 'inline' => true]
            ],
            'timestamp' => date('c'),
            'footer' => ['text' => 'WHMCS Notification']
        ];

        return $this->sendEmbed($embed, 'sales');
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
            'created' => ['color' => 0x3498db, 'title' => 'New Service Created'],
            'activated' => ['color' => 0x27ae60, 'title' => 'Service Activated'],
            'suspended' => ['color' => 0xe74c3c, 'title' => 'Service Suspended'],
            'terminated' => ['color' => 0x2c3e50, 'title' => 'Service Terminated']
        ];

        $config = $events[$event] ?? $events['created'];

        $embed = [
            'title' => $config['title'],
            'color' => $config['color'],
            'fields' => [
                ['name' => 'Service', 'value' => $service->domain, 'inline' => true],
                ['name' => 'Client', 'value' => $client['fullname'], 'inline' => true],
                ['name' => 'Status', 'value' => ucfirst($service->domainstatus), 'inline' => true],
                ['name' => 'Amount', 'value' => formatCurrency($service->amount), 'inline' => true]
            ],
            'timestamp' => date('c'),
            'footer' => ['text' => 'WHMCS Notification']
        ];

        return $this->sendEmbed($embed);
    }

    /**
     * Send support ticket notification
     */
    public function ticketNotification(int $ticketId, string $event): array
    {
        $ticket = Capsule::table('tbltickets')
            ->where('id', $ticketId)
            ->first();

        $priorityColors = ['Low' => 0x27ae60, 'Medium' => 0xf39c12, 'High' => 0xe67e22, 'Urgent' => 0xe74c3c];

        $events = [
            'created' => ['title' => 'New Support Ticket'],
            'replied' => ['title' => 'Ticket Reply'],
            'closed' => ['title' => 'Ticket Closed']
        ];

        $config = $events[$event] ?? $events['created'];

        $embed = [
            'title' => $config['title'],
            'color' => $priorityColors[$ticket->urgency] ?? 0x3498db,
            'fields' => [
                ['name' => 'Ticket', 'value' => "#{$ticketId}", 'inline' => true],
                ['name' => 'Subject', 'value' => $ticket->title, 'inline' => false],
                ['name' => 'Priority', 'value' => $ticket->urgency, 'inline' => true],
                ['name' => 'Status', 'value' => $ticket->status, 'inline' => true]
            ],
            'timestamp' => date('c'),
            'footer' => ['text' => 'WHMCS Support']
        ];

        return $this->sendEmbed($embed, 'support');
    }

    /**
     * Send error alert
     */
    public function errorAlert(string $title, string $message, array $context = []): array
    {
        $embed = [
            'title' => ":octagonal_sign: {$title}",
            'color' => 0xe74c3c,
            'description' => $message,
            'fields' => [],
            'timestamp' => date('c'),
            'footer' => ['text' => 'WHMCS Alert']
        ];

        foreach ($context as $key => $value) {
            $embed['fields'][] = ['name' => $key, 'value' => (string)$value, 'inline' => true];
        }

        return $this->sendEmbed($embed, 'alerts');
    }

    private function sendPayload(string $webhookUrl, array $payload): array
    {
        $ch = curl_init($webhookUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return ['success' => $response === '' || $response === 'ok'];
    }
}
```

### Step 2: Create Discord Hooks
```php
<?php
// /includes/hooks/discord_hooks.php

use WHMCS\Discord\DiscordManager;

$discordManager = new DiscordManager();

// Invoice events
add_hook('InvoicePaid', 1, function($vars) use ($discordManager) {
    $discordManager->invoiceNotification($vars['invoiceid'], 'paid');
});

// Service events
add_hook('ServiceSuspended', 1, function($vars) use ($discordManager) {
    $discordManager->serviceNotification($vars['serviceid'], 'suspended');
});

add_hook('ServiceTerminated', 1, function($vars) use ($discordManager) {
    $discordManager->serviceNotification($vars['serviceid'], 'terminated');
});

// Ticket events
add_hook('TicketOpen', 1, function($vars) use ($discordManager) {
    $discordManager->ticketNotification($vars['ticketid'], 'created');
});
```

## Related Workflows
- [WHMCS Slack Automation](./whmcs-slack-automation.md)
- [WHMCS Notification Automation](./whmcs-notification-automation.md)