# WHMCS Slack Integration Workflow

## Overview
This workflow implements Slack notifications for WHMCS.

## Prerequisites
- WHMCS with webhook access
- Slack workspace with webhook permissions

## Step-by-Step Process

### Step 1: Slack Integration
```php
<?php
// /includes/notifications/SlackNotification.php

class SlackNotification {
    private $webhookUrl;

    public function __construct()
    {
        $this->webhookUrl = getConfig('slack_webhook_url');
    }

    /**
     * Send notification
     */
    public function send(string $message, array $options = []): bool
    {
        $payload = [
            'text' => $message,
            'username' => $options['username'] ?? 'WHMCS Bot',
            'icon_emoji' => $options['icon'] ?? ':robot_face:'
        ];

        $ch = curl_init($this->webhookUrl);

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_HTTPHEADER => ['Content-Type: application/json']
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return $response === 'ok';
    }
}
```

### Step 2: Slack Hooks
```php
<?php
// /includes/hooks/slack_notifications.php

$slack = new SlackNotification();

add_hook('InvoicePaid', 1, function($vars) use ($slack) {
    $slack->send("Invoice #{$vars['invoiceid']} has been paid");
});
```

## Related Workflows
- [WHMCS Discord Integration](./whmcs-discord-integration.md)
- [WHMCS Notification Automation](./whmcs-notification-automation.md)