# WHMCS Discord Integration Workflow

## Overview
This workflow implements Discord notifications for WHMCS.

## Prerequisites
- WHMCS with webhook access
- Discord server with webhook permissions

## Step-by-Step Process

### Step 1: Discord Integration
```php
<?php
// /includes/notifications/DiscordNotification.php

class DiscordNotification {
    private $webhookUrl;

    public function __construct()
    {
        $this->webhookUrl = getConfig('discord_webhook_url');
    }

    /**
     * Send embed notification
     */
    public function sendEmbed(array $embed): bool
    {
        $payload = [
            'username' => 'WHMCS',
            'embeds' => [$embed]
        ];

        $ch = curl_init($this->webhookUrl);

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_HTTPHEADER => ['Content-Type: application/json']
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return empty($response);
    }

    /**
     * Send order notification
     */
    public function orderNotification(int $orderId): bool
    {
        $embed = [
            'title' => 'New Order',
            'color' => 0x00ff00,
            'fields' => [
                ['name' => 'Order ID', 'value' => (string)$orderId, 'inline' => true]
            ]
        ];

        return $this->sendEmbed($embed);
    }
}
```

## Related Workflows
- [WHMCS Slack Integration](./whmcs-slack-integration.md)
- [WHMCS Notification Automation](./whmcs-notification-automation.md)