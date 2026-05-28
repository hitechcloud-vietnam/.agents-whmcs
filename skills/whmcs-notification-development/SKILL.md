# WHMCS Notification Development Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for developing custom notification providers for WHMCS, including integration with external notification services, webhooks, and multi-channel notifications.

## When to Use

- Creating custom notification providers (Slack, Discord, SMS, etc.)
- Integrating with third-party messaging services
- Building multi-channel notification systems
- Setting up automated alert notifications

## Notification System Overview

### 1. WHMCS Notification Module Structure

```php
<?php
// modules/notifications/CustomProvider/

namespace WHMCS\Module\Notification\CustomProvider;

use WHMCS\Module\Contracts\NotificationModuleInterface;
use WHMCS\Notification\Contracts\NotificationInterface;
use WHMCS\Module\Notification\DescriptionTrait;

class Provider implements NotificationModuleInterface {
    use DescriptionTrait; // Required for all notification providers

    /**
     * Module configuration settings
     */
    public static function moduleConfiguration(): array {
        return [
            [
                'Name' => 'api_key',
                'Type' => 'password',
                'FriendlyName' => 'API Key',
                'Description' => 'Your notification service API key',
            ],
            [
                'Name' => 'channel',
                'Type' => 'text',
                'FriendlyName' => 'Default Channel',
                'Description' => 'Default channel for notifications',
            ],
            [
                'Name' => 'enabled_events',
                'Type' => 'dropdown',
                'FriendlyName' => 'Enabled Events',
                'Options' => [
                    'all' => 'All Events',
                    'critical' => 'Critical Only',
                    'billing' => 'Billing Events',
                ],
            ],
        ];
    }

    /**
     * Test connection to notification service
     * Must throw Exception on failure (NOT return false)
     */
    public function testConnection(): void {
        // Test API connection
        $apiKey = $this->getSetting('api_key');

        try {
            $response = $this->sendRequest('GET', '/test');

            if ($response['status'] !== 'ok') {
                throw new \Exception('Connection test failed');
            }
        } catch (\Exception $e) {
            throw new \Exception('Failed to connect: ' . $e->getMessage());
        }
    }

    /**
     * Define notification settings that appear in admin
     */
    public function notificationSettings(): array {
        return [
            [
                'Name' => 'channel_override',
                'Type' => 'text',
                'FriendlyName' => 'Channel Override',
                'Description' => 'Override the default channel for this notification',
            ],
            [
                'Name' => 'include_details',
                'Type' => 'yesno',
                'FriendlyName' => 'Include Full Details',
            ],
        ];
    }

    /**
     * Send the notification
     * Must throw Exception on failure
     */
    public function send(NotificationInterface $notification, array $settings): void {
        $channel = $settings['channel_override'] ?? $this->getSetting('channel');

        $message = $this->buildMessage($notification, $settings);

        try {
            $this->sendRequest('POST', '/messages', [
                'channel' => $channel,
                'text' => $message['text'],
                'embeds' => $message['embeds'],
            ]);
        } catch (\Exception $e) {
            throw new \Exception('Failed to send notification: ' . $e->getMessage());
        }
    }

    private function getSetting(string $name): mixed {
        return \WHMCS\Module\Setting::get($name);
    }

    private function buildMessage(NotificationInterface $notification, array $settings): array {
        $type = $notification->getType();

        $message = [
            'text' => $notification->getSubject(),
            'embeds' => [
                [
                    'title' => $notification->getSubject(),
                    'description' => $notification->getMessage(),
                    'color' => $this->getColorForType($type),
                    'fields' => [],
                ],
            ],
        ];

        // Add metadata fields
        if ($settings['include_details'] ?? false) {
            foreach ($notification->getProperties() as $key => $value) {
                $message['embeds'][0]['fields'][] = [
                    'name' => $key,
                    'value' => (string) $value,
                    'inline' => true,
                ];
            }
        }

        return $message;
    }

    private function getColorForType(string $type): int {
        $colors = [
            'invoice_created' => 0x3498db,    // Blue
            'invoice_paid' => 0x2ecc71,       // Green
            'invoice_overdue' => 0xe74c3c,    // Red
            'service_created' => 0x9b59b6,    // Purple
            'service_suspended' => 0xf39c12, // Orange
            'support_ticket' => 0x1abc9c,    // Teal
        ];

        return $colors[$type] ?? 0x95a5a6; // Gray default
    }

    private function sendRequest(string $method, string $endpoint, array $data = []): array {
        $apiKey = $this->getSetting('api_key');
        $baseUrl = 'https://api.notificationservice.com/v1';

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $baseUrl . $endpoint,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $apiKey,
                'Content-Type: application/json',
            ],
        ]);

        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return json_decode($response, true) ?? ['status' => 'error'];
    }
}
```

## Advanced Notification Patterns

### 1. Multi-Channel Notification System

```php
<?php
// includes/classes/MultiChannelNotification.php

namespace MyModule\Notifications;

use WHMCS\Notification\Contracts\NotificationInterface;

class MultiChannelNotifier {
    private array $channels = [];
    private array $eventFilters = [];

    public function __construct() {
        $this->loadConfiguredChannels();
    }

    private function loadConfiguredChannels(): void {
        $this->channels = [
            'email' => new EmailChannel(),
            'slack' => new SlackChannel(),
            'sms' => new SmsChannel(),
            'webhook' => new WebhookChannel(),
        ];

        $this->eventFilters = [
            'critical' => ['invoice_overdue', 'service_suspended', 'fraud_detected'],
            'billing' => ['invoice_paid', 'invoice_created', 'refund_processed'],
            'support' => ['ticket_opened', 'ticket_replied', 'ticket_escalated'],
        ];
    }

    public function notify(string $event, array $data): void {
        $channels = $this->getChannelsForEvent($event);

        foreach ($channels as $channelName) {
            if (!isset($this->channels[$channelName])) {
                continue;
            }

            try {
                $channel = $this->channels[$channelName];
                $channel->send($event, $data);
            } catch (\Exception $e) {
                logActivity("[MultiChannelNotifier] Failed to send via {$channelName}: " . $e->getMessage());
            }
        }
    }

    private function getChannelsForEvent(string $event): array {
        $channels = [];

        foreach ($this->eventFilters as $filterName => $events) {
            if (in_array($event, $events)) {
                $configuredChannels = $this->getChannelsForFilter($filterName);
                $channels = array_merge($channels, $configuredChannels);
            }
        }

        return array_unique($channels);
    }

    private function getChannelsForFilter(string $filter): array {
        return \WHMCS\Module\Notification::getSetting('filter_' . $filter . '_channels') ?? [];
    }

    // Channel implementations
    public function notifyInvoicePaid(int $invoiceId, float $amount): void {
        $invoice = \WHMCS\Database\Capsule::table('tblinvoices')->find($invoiceId);
        $client = \WHMCS\Database\Capsule::table('tblclients')->find($invoice->userid);

        $data = [
            'invoice_id' => $invoiceId,
            'invoice_number' => $invoice->invoicenum ?? $invoiceId,
            'amount' => formatCurrency($amount, $client->currency),
            'client_name' => $client->firstname . ' ' . $client->lastname,
            'client_email' => $client->email,
            'date' => date('Y-m-d H:i:s'),
        ];

        $this->notify('invoice_paid', $data);
    }

    public function notifyServiceSuspended(int $serviceId, string $reason): void {
        $service = \WHMCS\Database\Capsule::table('tblhosting')->find($serviceId);
        $client = \WHMCS\Database\Capsule::table('tblclients')->find($service->userid);

        $data = [
            'service_id' => $serviceId,
            'domain' => $service->domain,
            'client_name' => $client->firstname . ' ' . $client->lastname,
            'reason' => $reason,
            'date' => date('Y-m-d H:i:s'),
        ];

        $this->notify('service_suspended', $data);
    }
}

// Base channel interface
interface NotificationChannel {
    public function send(string $event, array $data): void;
}

// Email channel implementation
class EmailChannel implements NotificationChannel {
    public function send(string $event, array $data): void {
        $adminEmails = $this->getAdminEmails();

        foreach ($adminEmails as $email) {
            $this->sendEmail($email, $event, $data);
        }
    }

    private function getAdminEmails(): array {
        return \WHMCS\Database\Capsule::table(' tbladmins')
            ->where('disabled', 0)
            ->pluck('email')
            ->toArray();
    }

    private function sendEmail(string $to, string $event, array $data): void {
        // Send email implementation
    }
}

// Slack channel implementation
class SlackChannel implements NotificationChannel {
    public function send(string $event, array $data): void {
        $webhookUrl = \WHMCS\Module\Notification::getSetting('slack_webhook_url');

        if (!$webhookUrl) {
            return;
        }

        $message = $this->formatSlackMessage($event, $data);

        $this->sendWebhook($webhookUrl, $message);
    }

    private function formatSlackMessage(string $event, array $data): array {
        $emojis = [
            'invoice_paid' => ':moneybag:',
            'service_suspended' => ':warning:',
            'ticket_opened' => ':ticket:',
        ];

        return [
            'text' => $emojis[$event] ?? ':bell:',
            'attachments' => [
                [
                    'color' => $this->getColor($event),
                    'fields' => array_map(
                        fn($k, $v) => ['title' => $k, 'value' => $v, 'short' => true],
                        array_keys($data),
                        array_values($data)
                    ),
                ],
            ],
        ];
    }

    private function getColor(string $event): string {
        $colors = [
            'invoice_paid' => '#2ecc71',
            'service_suspended' => '#e74c3c',
        ];
        return $colors[$event] ?? '#95a5a6';
    }

    private function sendWebhook(string $url, array $payload): void {
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_RETURNTRANSFER => true,
        ]);
        curl_exec($ch);
        curl_close($ch);
    }
}

// SMS channel implementation
class SmsChannel implements NotificationChannel {
    public function send(string $event, array $data): void {
        $adminPhones = $this->getAdminPhones();

        foreach ($adminPhones as $phone) {
            $this->sendSms($phone, $event, $data);
        }
    }

    private function getAdminPhones(): array {
        return \WHMCS\Database\Capsule::table('tbladmins')
            ->where('disabled', 0)
            ->whereNotNull('mobile')
            ->pluck('mobile')
            ->toArray();
    }

    private function sendSms(string $phone, string $event, array $data): void {
        $message = $this->formatSmsMessage($event, $data);

        // Integrate with SMS provider (Twilio, etc.)
        $this->callSmsApi($phone, $message);
    }

    private function formatSmsMessage(string $event, array $data): string {
        $templates = [
            'invoice_paid' => "Invoice paid: {$data['invoice_number']} - {$data['amount']}",
            'service_suspended' => "Service suspended: {$data['domain']}",
        ];

        return $templates[$event] ?? "WHMSC Alert: {$event}";
    }

    private function callSmsApi(string $phone, string $message): void {
        // Twilio or other SMS API integration
    }
}

// Webhook channel implementation
class WebhookChannel implements NotificationChannel {
    public function send(string $event, array $data): void {
        $webhooks = $this->getConfiguredWebhooks();

        foreach ($webhooks as $webhook) {
            $this->sendWebhook($webhook, $event, $data);
        }
    }

    private function getConfiguredWebhooks(): array {
        return \WHMCS\Database\Capsule::table('mod_notification_webhooks')
            ->where('active', 1)
            ->whereJsonContains('events', $event)
            ->get()
            ->toArray();
    }

    private function sendWebhook(object $webhook, string $event, array $data): void {
        $payload = [
            'event' => $event,
            'timestamp' => date('c'),
            'data' => $data,
        ];

        $ch = curl_init($webhook->url);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-Webhook-Signature: ' . $this->generateSignature($payload, $webhook->secret),
            ],
        ]);
        curl_exec($ch);
        curl_close($ch);
    }

    private function generateSignature(array $payload, string $secret): string {
        return hash_hmac('sha256', json_encode($payload), $secret);
    }
}
```

### 2. Notification Template System

```php
<?php
// includes/classes/NotificationTemplateManager.php

namespace MyModule\Notifications;

class NotificationTemplateManager {
    private array $templates = [];

    public function __construct() {
        $this->loadTemplates();
    }

    private function loadTemplates(): void {
        $this->templates = [
            'invoice_paid' => [
                'email' => [
                    'subject' => 'Payment Received - Invoice #{invoice_number}',
                    'template' => 'invoice_payment_received',
                ],
                'slack' => [
                    'text' => ':moneybag: Payment received from {client_name}',
                    'fields' => ['amount', 'invoice_number'],
                ],
                'sms' => 'Invoice {invoice_number} paid: {amount}',
                'webhook' => [
                    'type' => 'payment_received',
                    'fields' => ['invoice_id', 'amount', 'client_email'],
                ],
            ],
            'service_suspended' => [
                'email' => [
                    'subject' => 'Service Suspended - {domain}',
                    'template' => 'service_suspended',
                ],
                'slack' => [
                    'text' => ':warning: Service suspended: {domain}',
                    'fields' => ['domain', 'reason'],
                ],
                'sms' => 'Service {domain} has been suspended',
                'webhook' => [
                    'type' => 'service_suspended',
                    'fields' => ['service_id', 'domain', 'reason'],
                ],
            ],
            'new_order' => [
                'email' => [
                    'subject' => 'New Order Received - #{order_id}',
                    'template' => 'new_order_admin',
                ],
                'slack' => [
                    'text' => ':shopping_cart: New order from {client_name}',
                    'fields' => ['order_id', 'amount', 'products'],
                ],
                'webhook' => [
                    'type' => 'new_order',
                    'fields' => ['order_id', 'client_id', 'items'],
                ],
            ],
        ];
    }

    public function render(string $channel, string $event, array $data): string {
        $template = $this->templates[$event][$channel] ?? null;

        if (!$template) {
            return '';
        }

        if (is_string($template)) {
            return $this->interpolate($template, $data);
        }

        if (isset($template['text'])) {
            return $this->interpolate($template['text'], $data);
        }

        return '';
    }

    private function interpolate(string $template, array $data): string {
        return preg_replace_callback('/\{(\w+)\}/', function($matches) use ($data) {
            return $data[$matches[1]] ?? $matches[0];
        }, $template);
    }

    public function getTemplateFields(string $event, string $channel): array {
        $template = $this->templates[$event][$channel] ?? [];
        return $template['fields'] ?? [];
    }
}
```

### 3. Notification Queue System

```php
<?php
// Notification queue for async delivery

class NotificationQueue {
    public static function add(string $channel, string $event, array $data, int $priority = 10): void {
        \WHMCS\Database\Capsule::table('mod_notification_queue')->insert([
            'channel' => $channel,
            'event' => $event,
            'payload' => json_encode($data),
            'priority' => $priority,
            'status' => 'pending',
            'attempts' => 0,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

// Process notification queue
add_hook('MinuteCronJob', 5, function($vars) {
    processNotificationQueue();
});

function processNotificationQueue(): void {
    $queue = \WHMCS\Database\Capsule::table('mod_notification_queue')
        ->where('status', 'pending')
        ->where('attempts', '<', 3)
        ->orderBy('priority', 'ASC')
        ->orderBy('created_at', 'ASC')
        ->limit(50)
        ->get();

    foreach ($queue as $item) {
        processQueueItem($item);
    }
}

function processQueueItem(object $item): void {
    \WHMCS\Database\Capsule::table('mod_notification_queue')
        ->where('id', $item->id)
        ->update(['status' => 'processing', 'attempts' => $item->attempts + 1]);

    try {
        $data = json_decode($item->payload, true);
        $notifier = new \MyModule\Notifications\MultiChannelNotifier();
        $notifier->notify($item->event, $data);

        \WHMCS\Database\Capsule::table('mod_notification_queue')
            ->where('id', $item->id)
            ->update(['status' => 'completed', 'completed_at' => date('Y-m-d H:i:s')]);

    } catch (\Exception $e) {
        $newAttempts = $item->attempts + 1;
        $status = $newAttempts >= 3 ? 'failed' : 'pending';

        \WHMCS\Database\Capsule::table('mod_notification_queue')
            ->where('id', $item->id)
            ->update([
                'status' => $status,
                'error' => $e->getMessage(),
                'next_attempt' => $status === 'pending'
                    ? date('Y-m-d H:i:s', time() + pow(2, $newAttempts) * 60)
                    : null,
            ]);
    }
}
```

## Notification Event Reference

| Event | Description | Available Properties |
|-------|-------------|---------------------|
| invoice_created | New invoice generated | invoice_id, user_id, total |
| invoice_paid | Invoice payment received | invoice_id, amount, trans_id |
| invoice_overdue | Invoice past due date | invoice_id, days_overdue |
| service_created | New service provisioned | service_id, domain |
| service_suspended | Service suspended | service_id, reason |
| service_unsuspended | Service reactivated | service_id |
| service_terminated | Service terminated | service_id, reason |
| ticket_opened | Support ticket created | ticket_id, subject |
| ticket_replied | Ticket reply received | ticket_id |
| order_placed | New order submitted | order_id |
| order_accepted | Order accepted/fulfilled | order_id |
| fraud_detected | Fraud check failed | order_id, score |

## Checklist

- [ ] Notification provider implements NotificationModuleInterface
- [ ] DescriptionTrait included
- [ ] moduleConfiguration() returns proper fields
- [ ] testConnection() throws Exception on failure
- [ ] notificationSettings() defined
- [ ] send() method properly implemented
- [ ] Error handling with proper exceptions
- [ ] Logging for debugging

---

**Related Skills:**
- whmcs-notification-builder
- whmcs-hooks-development
- whmcs-webhook-handler
- whmcs-sms-notification-builder

**Reference:**
- WHMCS Notification Modules: https://developers.whmcs.com/notifications/