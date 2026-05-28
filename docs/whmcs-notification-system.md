# WHMCS Notification System Reference

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-notification-builder`, `whmcs-sms-notification-builder`

---

## Overview

The WHMCS Notification system allows sending alerts and updates through various channels including SMS, Push notifications, Slack, Discord, Telegram, and custom providers. This reference covers the notification architecture, provider development, and integration patterns.

---

## Notification Architecture

### System Components

```
WHMCS Notification System
├── Notification Module (modules/notifications/)
│   ├── Provider implementations
│   ├── Template files
│   └── Assets (icons, logos)
├── Notification Rules (Setup > Automation Rules)
│   ├── Trigger events
│   ├── Recipient settings
│   └── Module-specific settings
└── Notification Events
    ├── Invoice events
    ├── Service events
    ├── Domain events
    └── Custom events
```

### Notification Module Interface

```php
<?php
namespace WHMCS\Module\Notification\MyProvider;

use WHMCS\Module\Contracts\NotificationModuleInterface;
use WHMCS\Notification\Contracts\NotificationInterface;
use WHMCS\Module\Notification\DescriptionTrait;

class Provider implements NotificationModuleInterface
{
    use DescriptionTrait; // Required - provides getName() and getDisplayName()

    /**
     * Module configuration fields
     */
    public static function moduleConfiguration(): array
    {
        return [
            [
                'Name' => 'api_key',
                'Type' => 'password',
                'FriendlyName' => 'API Key',
                'Description' => 'Enter your API key',
            ],
            [
                'Name' => 'default_channel',
                'Type' => 'text',
                'FriendlyName' => 'Default Channel',
            ],
            [
                'Name' => 'test_mode',
                'Type' => 'yesno',
                'FriendlyName' => 'Test Mode',
            ],
        ];
    }

    /**
     * Per-rule notification settings
     */
    public function notificationSettings(): array
    {
        return [
            [
                'Name' => 'channel_id',
                'Type' => 'text',
                'FriendlyName' => 'Channel/Recipient ID',
                'Description' => 'Channel or recipient for this rule',
            ],
        ];
    }

    /**
     * Test connection to notification service
     * @throws \Exception on failure - DO NOT return false
     */
    public function testConnection(): void
    {
        // Make API call to verify credentials
        $response = $this->api->ping();

        if (!$response->isOk()) {
            throw new \Exception('Failed to connect: ' . $response->error());
        }
    }

    /**
     * Send notification
     * @throws \Exception on failure - DO NOT return false
     */
    public function send(NotificationInterface $notification, array $settings): void
    {
        $channel = $settings['channel_id'] ?? $this->getDefaultChannel();

        $payload = $this->buildPayload($notification);

        $response = $this->api->send($channel, $payload);

        if (!$response->isOk()) {
            throw new \Exception('Failed to send: ' . $response->error());
        }
    }

    private function buildPayload(NotificationInterface $notification): array
    {
        return [
            'title' => $notification->getTitle(),
            'message' => $notification->getMessage(),
            'level' => $notification->getImportance(),
            'url' => $notification->getUrl(),
        ];
    }
}
```

---

## Notification Interface

### NotificationInterface Methods

```php
<?php
namespace WHMCS\Notification\Contracts;

interface NotificationInterface
{
    /**
     * Get notification unique ID
     */
    public function getId(): string;

    /**
     * Get notification title
     */
    public function getTitle(): string;

    /**
     * Get notification message/body
     */
    public function getMessage(): string;

    /**
     * Get notification importance level
     * @return string info|warning|error|success
     */
    public function getImportance(): string;

    /**
     * Get related URL (clickable in notification)
     */
    public function getUrl(): string;

    /**
     * Get notification timestamp
     */
    public function getDate(): \DateTime;

    /**
     * Get notification raw data
     */
    public function getRawData(): array;

    /**
     * Get associated entity type (client, invoice, ticket, etc.)
     */
    public function getEntityType(): ?string;

    /**
     * Get associated entity ID
     */
    public function getEntityId(): ?int;
}
```

---

## Notification Types

### Built-in Notification Classes

| Class | Description |
|-------|-------------|
| `WHMCS\Notification\Alert` | General alerts |
| `WHMCS\Notification\Invoice` | Invoice notifications |
| `WHMCS\Notification\Support` | Support ticket notifications |
| `WHMCS\Notification\Account` | Account notifications |
| `WHMCS\Notification\Domain` | Domain notifications |
| `WHMCS\Notification\Order` | Order notifications |

### Creating Custom Notifications

```php
<?php
use WHMCS\Notification\Alert;
use WHMCS\Module\Contracts\NotificationModuleInterface;

// Pre-built alert notification
$notification = new Alert(
    'Custom Alert Title',
    'This is the alert message',
    'warning'
);
$notification->setUrl('https://example.com/admin/view.php?id=123');
$notification->setEntity('service', $serviceId);

// Send via notification module
$modules = \WHMCS\Notification\Dependents::getAvailableNotificationModules();

foreach ($modules as $module) {
    $module->send($notification, $settings);
}
```

---

## Notification Module Development

### Module Structure

```
modules/notifications/MyProvider/
├── MyProvider.php          # Main module class
├── icon.png                 # 80x80px icon
└── README.md                # Documentation (optional)
```

### Complete Example

```php
<?php
/**
 * Slack Notification Provider
 */

namespace WHMCS\Module\Notification\Slack;

use WHMCS\Module\Contracts\NotificationModuleInterface;
use WHMCS\Notification\Contracts\NotificationInterface;
use WHMCS\Module\Notification\DescriptionTrait;

if (!defined('WHMCS')) {
    die('This file cannot be accessed directly');
}

class Slack implements NotificationModuleInterface
{
    use DescriptionTrait;

    private $httpClient;
    private $webhookUrl;
    private $botName = 'WHMCS';

    public static function moduleConfiguration(): array
    {
        return [
            [
                'Name' => 'webhook_url',
                'Type' => 'text',
                'FriendlyName' => 'Webhook URL',
                'Description' => 'Slack incoming webhook URL',
            ],
            [
                'Name' => 'bot_name',
                'Type' => 'text',
                'FriendlyName' => 'Bot Name',
                'Default' => 'WHMCS',
            ],
            [
                'Name' => 'icon_emoji',
                'Type' => 'text',
                'FriendlyName' => 'Icon (emoji)',
                'Default' => ':information_source:',
            ],
        ];
    }

    public function notificationSettings(): array
    {
        return [
            [
                'Name' => 'channel_override',
                'Type' => 'text',
                'FriendlyName' => 'Channel Override',
                'Description' => 'Override default channel for this rule',
            ],
        ];
    }

    public function testConnection(): void
    {
        $this->webhookUrl = trim(\WHMCS\Config\Setting::getValue('slack_webhook_url'));

        if (empty($this->webhookUrl)) {
            throw new \Exception('Webhook URL is not configured');
        }

        $response = $this->sendPayload([
            'text' => 'Test message from WHMCS',
            'username' => $this->botName,
        ]);

        if (!$response['ok']) {
            throw new \Exception('Webhook test failed: ' . ($response['error'] ?? 'Unknown error'));
        }
    }

    public function send(NotificationInterface $notification, array $settings): void
    {
        $this->webhookUrl = trim(\WHMCS\Config\Setting::getValue('slack_webhook_url'));

        $payload = $this->buildSlackPayload($notification, $settings);

        $response = $this->sendPayload($payload);

        if (!$response['ok']) {
            throw new \Exception('Failed to send: ' . ($response['error'] ?? 'Unknown error'));
        }
    }

    private function buildSlackPayload(NotificationInterface $notification, array $settings): array
    {
        $color = match($notification->getImportance()) {
            'error' => '#dc3545',
            'warning' => '#ffc107',
            'success' => '#28a745',
            default => '#17a2b8',
        };

        $message = [
            'username' => \WHMCS\Config\Setting::getValue('slack_bot_name') ?? $this->botName,
            'icon_emoji' => \WHMCS\Config\Setting::getValue('slack_icon_emoji') ?? $this->icon_emoji,
            'attachments' => [
                [
                    'fallback' => $notification->getTitle() . ': ' . $notification->getMessage(),
                    'color' => $color,
                    'title' => $notification->getTitle(),
                    'text' => $notification->getMessage(),
                    'fields' => [
                        [
                            'title' => 'Entity',
                            'value' => $notification->getEntityType() . ' #' . $notification->getEntityId(),
                            'short' => true,
                        ],
                        [
                            'title' => 'Date',
                            'value' => $notification->getDate()->format('Y-m-d H:i:s'),
                            'short' => true,
                        ],
                    ],
                ],
            ],
        ];

        if ($notification->getUrl()) {
            $message['attachments'][0]['title_link'] = $notification->getUrl();
        }

        return $message;
    }

    private function sendPayload(array $payload): array
    {
        $ch = curl_init($this->webhookUrl);

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true) ?? [];
    }
}
```

---

## Notification Events

### Available Event Types

| Category | Event | Trigger |
|----------|-------|---------|
| Invoice | `invoice_created` | New invoice generated |
| Invoice | `invoice_paid` | Payment received |
| Invoice | `invoice_overdue` | Invoice overdue |
| Invoice | `invoice_cancelled` | Invoice cancelled |
| Invoice | `invoice_refunded` | Invoice refunded |
| Service | `service_created` | New service provisioned |
| Service | `service_suspended` | Service suspended |
| Service | `service_terminated` | Service terminated |
| Service | `service_renewed` | Service renewed |
| Domain | `domain_expiring` | Domain expiring soon |
| Domain | `domain_expired` | Domain expired |
| Domain | `domain_transfer_declined` | Transfer declined |
| Order | `order_placed` | New order placed |
| Order | `order_accepted` | Order accepted |
| Order | `order_fraud` | Order flagged as fraud |
| Support | `ticket_opened` | New ticket created |
| Support | `ticket_reply` | Ticket replied |
| Support | `ticket_assigned` | Ticket assigned |
| Account | `password_reset` | Password reset requested |
| Account | `automatic_suspension` | Auto-suspension due to non-payment |
| Account | `automatic_termination` | Auto-termination due to non-payment |

---

## Notification Triggers

### Hook-based Triggers

```php
<?php
/**
 * Trigger custom notification
 */
function triggerCustomNotification(string $title, string $message, string $type = 'info'): void {
    $notification = new \WHMCS\Notification\Alert($title, $message, $type);

    // Get active notification modules
    $modules = \WHMCS\Notification\Dependents::getActiveNotificationModules();

    foreach ($modules as $moduleName) {
        try {
            $settings = getNotificationModuleSettings($moduleName);

            $provider = \WHMCS\Notification\Dependents::factory($moduleName);
            $provider->send($notification, $settings);

        } catch (\Exception $e) {
            logActivity("Notification failed: " . $e->getMessage());
        }
    }
}
```

### Trigger Examples

```php
<?php
// Trigger on custom event
add_hook('AfterModuleCreate', 1, function($vars) {
    triggerCustomNotification(
        'Service Activated',
        'Service #' . $vars['serviceid'] . ' has been activated',
        'success'
    );
});

// Custom payment notification
add_hook('InvoicePaid', 1, function($vars) {
    $invoice = Capsule::table('tblinvoices')
        ->where('id', $vars['invoiceid'])
        ->first();

    triggerCustomNotification(
        'Payment Received',
        "Invoice #{$invoice->invoicenum}: $" . number_format($vars['amount'], 2),
        'success'
    );
});
```

---

## Integration Patterns

### SMS Provider Example

```php
<?php
/**
 * SMS Notification Provider
 */
namespace WHMCS\Module\Notification\SmsProvider;

use WHMCS\Module\Contracts\NotificationModuleInterface;
use WHMCS\Notification\Contracts\NotificationInterface;
use WHMCS\Module\Notification\DescriptionTrait;

class Provider implements NotificationModuleInterface
{
    use DescriptionTrait;

    public static function moduleConfiguration(): array
    {
        return [
            ['Name' => 'api_key', 'Type' => 'password', 'FriendlyName' => 'API Key'],
            ['Name' => 'sender_id', 'Type' => 'text', 'FriendlyName' => 'Sender ID'],
        ];
    }

    public function notificationSettings(): array
    {
        return [
            ['Name' => 'phone_numbers', 'Type' => 'text', 'FriendlyName' => 'Phone Numbers'],
        ];
    }

    public function testConnection(): void
    {
        $response = $this->api->account();

        if (!$response->isOk()) {
            throw new \Exception('API connection failed');
        }
    }

    public function send(NotificationInterface $notification, array $settings): void
    {
        $phones = $this->getPhoneNumbers($settings);

        foreach ($phones as $phone) {
            $message = $notification->getTitle() . ': ' . $notification->getMessage();

            $response = $this->api->send([
                'to' => $phone,
                'message' => substr($message, 0, 160),
                'sender' => $this->getSenderId(),
            ]);

            if (!$response->isOk()) {
                logActivity("SMS send failed to {$phone}: " . $response->error());
            }
        }
    }

    private function getPhoneNumbers(array $settings): array
    {
        $phones = $settings['phone_numbers'] ?? '';

        return array_filter(array_map('trim', explode(',', $phones)));
    }
}
```

---

## Admin Notification Settings

### Per-Admin Notification Preferences

```php
<?php
/**
 * Get admin notification preferences
 */
function getAdminNotificationPrefs(int $adminId): array {
    return Capsule::table('mod_notification_prefs')
        ->where('admin_id', $adminId)
        ->pluck('setting_value', 'setting_key')
        ->toArray();
}

/**
 * Update admin notification preferences
 */
function updateAdminNotificationPrefs(int $adminId, array $prefs): void {
    foreach ($prefs as $key => $value) {
        Capsule::table('mod_notification_prefs')
            ->updateOrInsert(
                ['admin_id' => $adminId, 'setting_key' => $key],
                ['setting_value' => $value]
            );
    }
}
```

---

## Notification Queue

### Asynchronous Notification Processing

```php
<?php
/**
 * Queue notification for async processing
 */
function queueNotification(array $notificationData, int $delay = 0): void {
    Capsule::table('mod_notification_queue')->insert([
        'notification_type' => $notificationData['type'],
        'notification_data' => json_encode($notificationData),
        'status' => 'pending',
        'delay_until' => $delay > 0 ? date('Y-m-d H:i:s', time() + $delay) : null,
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Process notification queue
 */
function processNotificationQueue(): void {
    $pending = Capsule::table('mod_notification_queue')
        ->where('status', 'pending')
        ->whereRaw("delay_until IS NULL OR delay_until <= NOW()")
        ->limit(100)
        ->get();

    foreach ($pending as $item) {
        try {
            $data = json_decode($item->notification_data, true);

            $notification = new \WHMCS\Notification\Alert(
                $data['title'],
                $data['message'],
                $data['type'] ?? 'info'
            );

            $modules = \WHMCS\Notification\Dependents::getActiveNotificationModules();

            foreach ($modules as $module) {
                $module->send($notification, getNotificationModuleSettings($module));
            }

            Capsule::table('mod_notification_queue')
                ->where('id', $item->id)
                ->update(['status' => 'sent']);

        } catch (\Exception $e) {
            Capsule::table('mod_notification_queue')
                ->where('id', $item->id)
                ->update([
                    'status' => 'failed',
                    'error' => $e->getMessage(),
                ]);
        }
    }
}
```

---

## Best Practices

1. **Use trait** - Always use `DescriptionTrait` for consistent naming
2. **Throw exceptions** - Never return `false` on errors; throw `\Exception`
3. **Test connection** - Implement `testConnection()` for admin verification
4. **Sanitize input** - Validate all user-configured values
5. **Queue large sends** - Process bulk rates asynchronously
6. **Log failures** - Record failed sends for debugging
7. **Support retry** - Design for idempotent operations
8. **Rate limiting** - Respect API rate limits

---

## Related Documentation

- [Notification Builder Skill](../skills/whmcs-notification-builder)
- [SMS Notification Builder](../skills/whmcs-sms-notification-builder)
- [Notification Developer Guide](notification-developer-guide.md)
