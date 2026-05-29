# WHMCS Notification Provider Development

## Overview

Notification providers allow WHMCS to send notifications through various channels like SMS, Slack, Discord, and custom integrations.

## Module Structure

```
modules/notifications/YourProvider/
├── NotificationProvider.php    # Main provider class
└── lang/
    └── english.php
```

## Provider Class

```php
<?php
// modules/notifications/YourProvider/NotificationProvider.php

namespace WHMCS\Module\Notification\YourProvider;

use WHMCS\Module\Contracts\NotificationModuleInterface;
use WHMCS\Notification\Contracts\NotificationInterface;
use WHMCS\Module\Notification\DescriptionTrait;

class NotificationProvider implements NotificationModuleInterface
{
    use DescriptionTrait;
    
    public static function moduleConfiguration(): array
    {
        return [
            [
                'Name' => 'Webhook URL',
                'Type' => 'text',
                'FriendlyName' => 'Webhook URL',
                'Description' => 'Enter your notification webhook URL',
            ],
            [
                'Name' => 'Bot Token',
                'Type' => 'password',
                'FriendlyName' => 'Bot Token',
                'Description' => 'Your bot authentication token',
            ],
            [
                'Name' => 'Channel ID',
                'Type' => 'text',
                'FriendlyName' => 'Channel ID',
                'Description' => 'Target channel ID',
            ],
        ];
    }
    
    public function testConnection(): void
    {
        $webhookUrl = $this->getSetting('webhook_url');
        
        if (empty($webhookUrl)) {
            throw new \Exception('Webhook URL is required');
        }
        
        // Test connection
        $ch = curl_init($webhookUrl);
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 10,
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        if ($httpCode >= 400) {
            throw new \Exception('Failed to connect to webhook URL');
        }
    }
    
    public function notificationSettings(): array
    {
        return [
            [
                'Name' => 'channel_override',
                'Type' => 'text',
                'FriendlyName' => 'Channel Override',
                'Description' => 'Override default channel for this notification',
                'Required' => false,
            ],
        ];
    }
    
    public function send(NotificationInterface $notification, array $settings): void
    {
        $webhookUrl = $settings['webhook_url'] ?? '';
        $channelId = $settings['channel_id'] ?? $settings['channel_override'] ?? '';
        
        if (empty($webhookUrl)) {
            throw new \Exception('Webhook URL not configured');
        }
        
        $payload = $this->buildPayload($notification, $channelId);
        
        $this->sendWebhook($webhookUrl, $payload);
    }
    
    private function buildPayload(NotificationInterface $notification, string $channelId): array
    {
        $type = $notification->getType();
        $title = $notification->getTitle();
        $message = $notification->getMessage();
        $metadata = $notification->getMetadata();
        
        $embed = [
            'title' => $title,
            'description' => $message,
            'color' => $this->getColorForType($type),
            'fields' => [],
            'footer' => [
                'text' => 'WHMCS Notification',
            ],
            'timestamp' => date('c'),
        ];
        
        // Add metadata as fields
        foreach ($metadata as $key => $value) {
            if (is_scalar($value)) {
                $embed['fields'][] = [
                    'name' => ucfirst(str_replace('_', ' ', $key)),
                    'value' => (string) $value,
                    'inline' => true,
                ];
            }
        }
        
        return [
            'channel_id' => $channelId,
            'embeds' => [$embed],
        ];
    }
    
    private function getColorForType(string $type): int
    {
        $colors = [
            'info' => 3447003,      // Blue
            'success' => 3066993,   // Green
            'warning' => 15105570, // Orange
            'error' => 15158332,    // Red
        ];
        
        return $colors[$type] ?? 0;
    }
    
    private function sendWebhook(string $url, array $payload): void
    {
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
            ],
            CURLOPT_TIMEOUT => 30,
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        if ($httpCode >= 400) {
            throw new \Exception("Webhook failed with HTTP code {$httpCode}: {$response}");
        }
    }
    
    private function getSetting(string $name): ?string
    {
        return \WHMCS\Config\Setting::getValue('modNotification_' . $name);
    }
}
```

## Slack Integration Example

```php
<?php
namespace WHMCS\Module\Notification\Slack;

use WHMCS\Module\Contracts\NotificationModuleInterface;
use WHMCS\Notification\Contracts\NotificationInterface;
use WHMCS\Module\Notification\DescriptionTrait;

class Slack implements NotificationModuleInterface
{
    use DescriptionTrait;
    
    public static function moduleConfiguration(): array
    {
        return [
            [
                'Name' => 'webhook_url',
                'Type' => 'text',
                'FriendlyName' => 'Webhook URL',
                'Description' => 'Your Slack incoming webhook URL',
            ],
            [
                'Name' => 'username',
                'Type' => 'text',
                'FriendlyName' => 'Bot Username',
                'Default' => 'WHMCS Bot',
            ],
            [
                'Name' => 'icon_emoji',
                'Type' => 'text',
                'FriendlyName' => 'Icon Emoji',
                'Default' => ':bell:',
            ],
        ];
    }
    
    public function testConnection(): void
    {
        $webhookUrl = Setting::getValue('modNotificationSlack_webhook_url');
        
        if (empty($webhookUrl)) {
            throw new \Exception('Webhook URL is required');
        }
    }
    
    public function notificationSettings(): array
    {
        return [
            [
                'Name' => 'include_fields',
                'Type' => 'yesno',
                'FriendlyName' => 'Include Notification Fields',
                'Description' => 'Include detailed fields in the notification',
            ],
        ];
    }
    
    public function send(NotificationInterface $notification, array $settings): void
    {
        $webhookUrl = $settings['webhook_url'];
        
        $payload = [
            'username' => $settings['username'] ?? 'WHMCS',
            'icon_emoji' => $settings['icon_emoji'] ?? ':bell:',
            'attachments' => [
                [
                    'color' => $this->getColor($notification->getType()),
                    'title' => $notification->getTitle(),
                    'text' => $notification->getMessage(),
                    'fields' => [],
                    'footer' => 'WHMCS',
                    'ts' => time(),
                ],
            ],
        ];
        
        if (!empty($settings['include_fields'])) {
            foreach ($notification->getMetadata() as $key => $value) {
                if (is_scalar($value)) {
                    $payload['attachments'][0]['fields'][] = [
                        'title' => ucfirst(str_replace('_', ' ', $key)),
                        'value' => (string) $value,
                        'short' => strlen((string) $value) < 30,
                    ];
                }
            }
        }
        
        $this->postToSlack($webhookUrl, $payload);
    }
    
    private function getColor(string $type): string
    {
        return match ($type) {
            'info' => '#3498db',
            'success' => '#2ecc71',
            'warning' => '#f39c12',
            'error' => '#e74c3c',
            default => '#95a5a6',
        };
    }
    
    private function postToSlack(string $webhookUrl, array $payload): void
    {
        $ch = curl_init($webhookUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
    }
}
```

## SMS Provider Example

```php
<?php
namespace WHMCS\Module\Notification\Twilio;

use WHMCS\Module\Contracts\NotificationModuleInterface;
use WHMCS\Notification\Contracts\NotificationInterface;
use WHMCS\Module\Notification\DescriptionTrait;

class Twilio implements NotificationModuleInterface
{
    use DescriptionTrait;
    
    public static function moduleConfiguration(): array
    {
        return [
            ['Name' => 'account_sid', 'Type' => 'text', 'FriendlyName' => 'Account SID'],
            ['Name' => 'auth_token', 'Type' => 'password', 'FriendlyName' => 'Auth Token'],
            ['Name' => 'from_number', 'Type' => 'text', 'FriendlyName' => 'From Number'],
        ];
    }
    
    public function testConnection(): void
    {
        $sid = Setting::getValue('modNotificationTwilio_account_sid');
        if (empty($sid)) {
            throw new \Exception('Account SID is required');
        }
    }
    
    public function notificationSettings(): array
    {
        return [
            ['Name' => 'admin_phone', 'Type' => 'text', 'FriendlyName' => 'Admin Phone Number'],
        ];
    }
    
    public function send(NotificationInterface $notification, array $settings): void
    {
        $sid = $settings['account_sid'];
        $token = $settings['auth_token'];
        $from = $settings['from_number'];
        $to = $settings['admin_phone'];
        
        $message = $notification->getTitle() . "\n" . $notification->getMessage();
        
        $this->sendSMS($sid, $token, $from, $to, $message);
    }
    
    private function sendSMS(string $sid, string $token, string $from, string $to, string $message): void
    {
        $url = "https://api.twilio.com/2010-04-01/Accounts/{$sid}/Messages.json";
        
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_USERPWD => "{$sid}:{$token}",
            CURLOPT_POSTFIELDS => [
                'From' => $from,
                'To' => $to,
                'Body' => $message,
            ],
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        $result = json_decode($response, true);
        
        if (isset($result['error'])) {
            throw new \Exception($result['error']['message']);
        }
    }
}
```

## Best Practices

1. **Throw exceptions on failure** - Don't return false
2. **Implement testConnection** - Allow users to verify setup
3. **Include metadata fields** - Provide detailed information
4. **Handle timeouts** - Use appropriate timeouts
5. **Support custom channels** - Allow per-notification overrides

## Related Documentation

- [WHMCS Notification System](/docs/whmcs-notification-system.md)