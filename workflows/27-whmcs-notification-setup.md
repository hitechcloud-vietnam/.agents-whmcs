# WHMCS Notification Setup Workflow

## Overview
This workflow covers setting up custom notifications for WHMCS events.

## Step 1: Notification Service

```php
<?php
// src/Service/NotificationService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class NotificationService
{
    private $channels = [];
    private $templates = [];

    public function registerChannel(string $name, callable $handler): void
    {
        $this->channels[$name] = $handler;
    }

    public function send(string $channel, string $template, array $data): array
    {
        if (!isset($this->channels[$channel])) {
            throw new \Exception("Channel not registered: $channel");
        }

        $handler = $this->channels[$channel];
        $message = $this->renderTemplate($template, $data);

        $result = $handler($message, $data);

        // Log notification
        $this->logNotification($channel, $template, $data, $result);

        return $result;
    }

    public function sendToClient(int $clientId, string $channel, string $template, array $data): void
    {
        $client = Capsule::table('tblclients')->where('id', $clientId)->first();
        $data['client'] = $client;
        $data['client_email'] = $client->email;

        $this->send($channel, $template, $data);
    }

    private function renderTemplate(string $template, array $data): array
    {
        $templates = $this->getTemplates();

        if (!isset($templates[$template])) {
            throw new \Exception("Template not found: $template");
        }

        $tpl = $templates[$template];

        return [
            'subject' => $this->interpolate($tpl['subject'], $data),
            'body' => $this->interpolate($tpl['body'], $data)
        ];
    }

    private function interpolate(string $text, array $data): string
    {
        foreach ($data as $key => $value) {
            if (is_scalar($value)) {
                $text = str_replace('{{' . $key . '}}', (string)$value, $text);
            }
        }
        return $text;
    }

    private function logNotification(string $channel, string $template, array $data, $result): void
    {
        Capsule::table('mod_notifications')->insert([
            'channel' => $channel,
            'template' => $template,
            'recipient' => $data['client_email'] ?? json_encode($data),
            'status' => $result['success'] ?? false ? 'sent' : 'failed',
            'error' => $result['error'] ?? null,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    private function getTemplates(): array
    {
        return [
            'service_expiring' => [
                'subject' => 'Service Expiring Soon - {{service_name}}',
                'body' => 'Your service {{service_name}} will expire on {{expiry_date}}.'
            ],
            'payment_overdue' => [
                'subject' => 'Payment Overdue - Invoice #{{invoice_id}}',
                'body' => 'Your invoice #{{invoice_id}} for {{amount}} is {{days_overdue}} days overdue.'
            ],
            'welcome' => [
                'subject' => 'Welcome to {{company_name}}!',
                'body' => 'Thank you for joining us, {{client_name}}!'
            ]
        ];
    }
}
```

## Step 2: Notification Channels

```php
<?php
// includes/hooks/notification_channels.php

use WHMCS\Module\Addon\YourModule\Service\NotificationService;

$notificationService = new NotificationService();

// Email channel (default)
$notificationService->registerChannel('email', function($message, $data) {
    if (isset($data['client_email'])) {
        send_email($data['template'] ?? 'General', $data['client_email'], [
            'subject' => $message['subject'],
            'body' => $message['body']
        ]);
        return ['success' => true];
    }
    return ['success' => false, 'error' => 'No email provided'];
});

// Slack channel
$notificationService->registerChannel('slack', function($message, $data) {
    $webhookUrl = Capsule::config('slack_webhook_url');

    $ch = curl_init($webhookUrl);
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode([
            'text' => $message['subject'],
            'blocks' => [
                ['type' => 'section', 'text' => ['type' => 'mrkdwn', 'text' => $message['body']]]
            ]
        ]),
        CURLOPT_RETURNTRANSFER => true
    ]);
    curl_exec($ch);
    curl_close($ch);

    return ['success' => true];
});

// SMS channel
$notificationService->registerChannel('sms', function($message, $data) {
    if (!isset($data['phone'])) {
        return ['success' => false, 'error' => 'No phone number'];
    }

    // Send SMS via provider
    $provider = Capsule::config('sms_provider');
    $apiKey = Capsule::config('sms_api_key');

    $ch = curl_init("$provider/send");
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode([
            'to' => $data['phone'],
            'message' => $message['body']
        ]),
        CURLOPT_HTTPHEADER => ['Authorization: Bearer ' . $apiKey],
        CURLOPT_RETURNTRANSFER => true
    ]);
    curl_exec($ch);
    curl_close($ch);

    return ['success' => true];
});
```

## Verification Checklist

- [ ] Notification service implemented
- [ ] Channels registered
- [ ] Templates configured
- [ ] Logging working
- [ ] Test notification sent successfully
