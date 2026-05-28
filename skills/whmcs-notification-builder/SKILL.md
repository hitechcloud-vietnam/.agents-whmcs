# WHMCS Notification Provider Builder Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building WHMCS notification provider modules for SMS, push notifications, and chat integrations.

## When to Use

- Creating SMS notification providers
- Adding push notification support (FCM, OneSignal)
- Integrating chat platforms (Zalo, Telegram)
- Building custom notification channels

## Structure

```
modules/notifications/{Provider}/
├── {Provider}.php            ← Main notification file
├── logo.png                  ← 80x80px logo
└── whmcs.json               ← Module metadata
```

## Building Steps

### Step 1: Create Notification Provider

```php
<?php
// modules/notifications/{Provider}/{Provider}.php
namespace WHMCS\Module\Notification;

use WHMCS\Module\Contracts\NotificationModuleInterface;
use WHMCS\Notification\Contracts\NotificationInterface;

if (!defined("WHMCS")) { die("Direct access denied"); }

class {Provider} implements NotificationModuleInterface
{
    use DescriptionTrait;

    public static function moduleConfiguration(): array {
        return [
            ['Name' => 'api_key', 'Type' => 'password', 'FriendlyName' => 'API Key'],
            ['Name' => 'api_secret', 'Type' => 'password', 'FriendlyName' => 'API Secret'],
            ['Name' => 'sender_name', 'Type' => 'text', 'FriendlyName' => 'Sender Name'],
        ];
    }

    public function testConnection(): void {
        $apiKey = $this->getSetting('api_key');
        $apiSecret = $this->getSetting('api_secret');

        if (empty($apiKey) || empty($apiSecret)) {
            throw new \Exception('API credentials not configured');
        }

        try {
            $response = $this->apiRequest('GET', '/account/verify');
            if (!$response['success']) {
                throw new \Exception('Invalid API credentials');
            }
        } catch (\Exception $e) {
            throw new \Exception('Connection failed: ' . $e->getMessage());
        }
    }

    public function notificationSettings(): array {
        return [
            ['Name' => 'recipient_type', 'Type' => 'dropdown', 'FriendlyName' => 'Recipient',
             'Options' => ['client_mobile' => 'Client Mobile', 'admin_mobile' => 'Admin Mobile', 'static' => 'Static Number']],
            ['Name' => 'static_recipient', 'Type' => 'text', 'FriendlyName' => 'Static Number'],
            ['Name' => 'template', 'Type' => 'textarea', 'FriendlyName' => 'Message Template',
             'Description' => 'Available: {title}, {message}, {url}'],
        ];
    }

    public function send(NotificationInterface $notification, array $settings): void {
        $recipient = $this->resolveRecipient($settings);
        if (empty($recipient)) {
            throw new \Exception('No recipient configured');
        }

        $body = str_replace(
            ['{title}', '{message}', '{url}'],
            [$notification->getTitle(), $notification->getMessage(), $notification->getUrl()],
            $settings['template'] ?? '{title}: {message}'
        );

        try {
            $result = $this->apiRequest('POST', '/messages/send', [
                'to' => $recipient,
                'message' => $body,
                'from' => $this->getSetting('sender_name'),
            ]);

            if (!$result['success']) {
                throw new \Exception($result['error'] ?? 'Send failed');
            }
        } catch (\Exception $e) {
            throw new \Exception('Send failed: ' . $e->getMessage());
        }
    }

    public function getDynamicField(string $fieldName, array $settings): array {
        return [];
    }

    public function getDisplayName(): string {
        return '{Provider} Notifications';
    }

    public function getLogoFileName(): string {
        return 'logo.png';
    }

    private function getSetting(string $name): string {
        return \Config::getInstance()->get('modNotifications_' . static::class . '_' . $name) ?? '';
    }

    private function resolveRecipient(array $settings): string {
        return match ($settings['recipient_type'] ?? 'client_mobile') {
            'client_mobile' => $settings['client_mobile'] ?? '',
            'admin_mobile' => $settings['admin_mobile'] ?? '',
            'static' => $settings['static_recipient'] ?? '',
            default => '',
        };
    }

    private function apiRequest(string $method, string $endpoint, array $data = []): array {
        $apiKey = $this->getSetting('api_key');

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => 'https://api.provider.com' . $endpoint,
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
        curl_close($ch);

        return json_decode($response, true) ?? [];
    }
}
```

### Step 2: Create whmcs.json

```json
{
    "schema": "1.0",
    "type": "notification-provider",
    "name": { "en": "{Provider} Notifications" },
    "description": { "en": "Send notifications via {Provider}" },
    "version": "1.0.0",
    "authors": [{ "name": "HiTechCloud", "homepage": "https://hitechcloud.vn" }],
    "categories": ["notifications", "sms"]
}
```

## Vietnamese Providers

### Zalo ZBS Pattern
```php
// Zalo ZBS notification
public function send(NotificationInterface $notification, array $settings): void {
    $accessToken = $this->getAccessToken();
    $recipient = $this->resolveRecipient($settings);

    $response = $this->callZaloApi('/zsender/phone/template', $accessToken, [
        'phone' => $recipient,
        'template_id' => $settings['template_id'],
        'template_data' => [
            'customer_name' => $notification->getTitle(),
            'message' => $notification->getMessage(),
        ],
    ]);

    if (!$response['success']) {
        throw new \Exception('Zalo send failed');
    }
}
```

### FCM Pattern
```php
// Firebase Cloud Messaging
public function send(NotificationInterface $notification, array $settings): void {
    $projectId = $this->getSetting('project_id');
    $deviceToken = $settings['device_token'];

    $response = $this->sendFCM($projectId, $deviceToken, [
        'title' => $notification->getTitle(),
        'body' => $notification->getMessage(),
    ]);

    if (!$response['success']) {
        throw new \Exception('FCM send failed');
    }
}
```

## Checklist

- [ ] use DescriptionTrait (REQUIRED)
- [ ] implements NotificationModuleInterface
- [ ] moduleConfiguration() returns array
- [ ] testConnection() throws Exception
- [ ] notificationSettings() returns per-rule settings
- [ ] send() throws Exception on failure
- [ ] getDisplayName() returns provider name
- [ ] getLogoFileName() returns 'logo.png'
- [ ] logo.png (80x80px)
- [ ] whmcs.json

---

**Related Skills:**
- whmcs-notification-sms
- whmcs-notification-push
- whmcs-notification-testing