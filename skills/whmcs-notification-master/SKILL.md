# WHMCS Notification Provider Master

## Overview
Master skill for WHMCS notification provider development. Covers custom notification channels, provider implementation, message formatting, and webhook integration.

## Notification Provider Structure

```php
<?php
// /modules/notifications/YourNotification/YourNotification.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Module\Notification\YourNotification\Provider;

return [
    'name' => 'Your Notification Provider',
    'description' => 'Description of the notification provider',
    'version' => '1.0.0',
    'author' => 'Your Name',
    'fields' => [
        'webhookUrl' => [
            'Name' => 'Webhook URL',
            'Type' => 'text',
            'Required' => true,
            'Description' => 'Enter your webhook URL',
        ],
        'apiKey' => [
            'Name' => 'API Key',
            'Type' => 'password',
            'Required' => false,
            'Description' => 'Optional API key for authentication',
        ],
        'channel' => [
            'Name' => 'Channel/Room',
            'Type' => 'text',
            'Required' => false,
            'Description' => 'Optional channel or room ID',
        ],
        'username' => [
            'Name' => 'Username',
            'Type' => 'text',
            'Required' => false,
            'Description' => 'Username for the notification bot',
        ],
        'template' => [
            'Name' => 'Message Template',
            'Type' => 'dropdown',
            'Options' => [
                'simple' => 'Simple Text',
                'rich' => 'Rich Format',
                'full' => 'Full Details',
            ],
            'Default' => 'rich',
        ],
        'debugMode' => [
            'Name' => 'Debug Mode',
            'Type' => 'yesno',
            'Description' => 'Enable debug logging',
        ],
    ],
];
```

## Provider Implementation

```php
<?php
// /modules/notifications/YourNotification/Provider.php

namespace WHMCS\Module\Notification\YourNotification;

use WHMCS\Module\Contracts\NotificationModuleInterface;
use WHMCS\Module\Notification\AbstractNotificationModule;
use WHMCS\Exception\Module\InvalidConfiguration;

class Provider extends AbstractNotificationModule implements NotificationModuleInterface
{
    protected $webhookUrl;
    protected $apiKey;
    protected $channel;
    protected $username;
    protected $template;
    protected $debugMode;

    public function __construct()
    {
        // Set notification method display names
        $this->setNotificationName('YourNotification', 'Your Notification Service');

        // Define available notification methods
        $this->addNotificationMethod(
            'general',
            [
                'name' => 'General Notifications',
                'description' => 'Receive general WHMCS notifications',
                'icon_class' => 'fa-bell',
            ]
        );

        $this->addNotificationMethod(
            'support',
            [
                'name' => 'Support Tickets',
                'description' => 'Receive support ticket notifications',
                'icon_class' => 'fa-life-ring',
            ]
        );

        $this->addNotificationMethod(
            'billing',
            [
                'name' => 'Billing Alerts',
                'description' => 'Receive billing and invoice notifications',
                'icon_class' => 'fa-credit-card',
            ]
        );

        $this->addNotificationMethod(
            'orders',
            [
                'name' => 'New Orders',
                'description' => 'Receive new order notifications',
                'icon_class' => 'fa-shopping-cart',
            ]
        );
    }

    public function validateConfiguration(array $settings): array
    {
        if (empty($settings['webhookUrl'])) {
            return [
                'success' => false,
                'error' => 'Webhook URL is required',
            ];
        }

        if (!filter_var($settings['webhookUrl'], FILTER_VALIDATE_URL)) {
            return [
                'success' => false,
                'error' => 'Invalid webhook URL format',
            ];
        }

        // Test the connection
        try {
            $result = $this->sendTestNotification($settings);

            if ($result) {
                return ['success' => true];
            }

            return [
                'success' => false,
                'error' => 'Failed to connect to webhook URL',
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => 'Connection error: ' . $e->getMessage(),
            ];
        }
    }

    public function sendTestNotification(array $settings): bool
    {
        $payload = [
            'text' => 'This is a test notification from WHMCS',
            'attachments' => [
                [
                    'title' => 'Test Notification',
                    'text' => 'Your notification provider is configured correctly!',
                    'color' => '#00ff00',
                ],
            ],
        ];

        return $this->sendWebhook($settings, $payload);
    }

    public function sendNotification(array $settings, string $method, string $title, string $body, array $data = [], array $attributes = [])
    {
        $this->webhookUrl = $settings['webhookUrl'];
        $this->apiKey = $settings['apiKey'] ?? '';
        $this->channel = $settings['channel'] ?? '';
        $this->username = $settings['username'] ?? 'WHMCS Bot';
        $this->template = $settings['template'] ?? 'rich';
        $this->debugMode = $settings['debugMode'] ?? false;

        $payload = $this->formatPayload($method, $title, $body, $data, $attributes);

        if ($this->debugMode) {
            logActivity("[YourNotification] Sending notification: " . json_encode($payload));
        }

        return $this->sendWebhook($settings, $payload);
    }

    protected function formatPayload(string $method, string $title, string $body, array $data, array $attributes): array
    {
        switch ($this->template) {
            case 'simple':
                return $this->formatSimplePayload($title, $body);

            case 'full':
                return $this->formatFullPayload($method, $title, $body, $data, $attributes);

            case 'rich':
            default:
                return $this->formatRichPayload($method, $title, $body, $data, $attributes);
        }
    }

    protected function formatSimplePayload(string $title, string $body): array
    {
        return [
            'text' => $title . "\n" . $body,
        ];
    }

    protected function formatRichPayload(string $method, string $title, string $body, array $data, array $attributes): array
    {
        $color = $this->getColorForMethod($method);

        return [
            'username' => $this->username,
            'channel' => $this->channel ?: null,
            'text' => $title,
            'attachments' => [
                [
                    'title' => $title,
                    'text' => $body,
                    'color' => $color,
                    'footer' => 'WHMCS Notification',
                    'footer_icon' => 'https://docs.whmcs.com/favicon.ico',
                    'ts' => time(),
                ],
            ],
        ];
    }

    protected function formatFullPayload(string $method, string $title, string $body, array $data, array $attributes): array
    {
        $color = $this->getColorForMethod($method);
        $fields = [];

        foreach ($attributes as $key => $value) {
            $fields[] = [
                'title' => ucfirst(str_replace('_', ' ', $key)),
                'value' => (string)$value,
                'short' => strlen((string)$value) < 40,
            ];
        }

        foreach ($data as $key => $value) {
            $fields[] = [
                'title' => ucfirst(str_replace('_', ' ', $key)),
                'value' => is_array($value) ? json_encode($value) : (string)$value,
                'short' => strlen((string)$value) < 40,
            ];
        }

        return [
            'username' => $this->username,
            'channel' => $this->channel ?: null,
            'attachments' => [
                [
                    'title' => $title,
                    'text' => $body,
                    'color' => $color,
                    'fields' => $fields,
                    'footer' => 'WHMCS Notification - ' . date('Y-m-d H:i:s'),
                    'ts' => time(),
                ],
            ],
        ];
    }

    protected function getColorForMethod(string $method): string
    {
        $colors = [
            'general' => '#3498db',    // Blue
            'support' => '#9b59b6',   // Purple
            'billing' => '#2ecc71',   // Green
            'orders' => '#e67e22',    // Orange
            'critical' => '#e74c3c',  // Red
            'warning' => '#f39c12',  // Yellow
        ];

        return $colors[$method] ?? '#95a5a6';
    }

    protected function sendWebhook(array $settings, array $payload): bool
    {
        $headers = [
            'Content-Type: application/json',
            'Accept: application/json',
        ];

        if (!empty($settings['apiKey'])) {
            $headers[] = 'Authorization: Bearer ' . $settings['apiKey'];
        }

        $ch = curl_init($settings['webhookUrl']);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_HTTPHEADER => $headers,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_SSL_VERIFYPEER => true,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception('Webhook error: ' . $error);
        }

        if ($httpCode >= 400) {
            throw new \Exception('Webhook returned HTTP ' . $httpCode);
        }

        return $httpCode >= 200 && $httpCode < 300;
    }

    // Handle specific notification events
    public function notificationAvailable(string $notificationName): bool
    {
        $available = [
            'ticket_open',
            'ticket_reply',
            'ticket_close',
            'ticket_note',
            'invoice_created',
            'invoice_paid',
            'invoice_overdue',
            'order_new',
            'order_pending',
            'order_accepted',
            'order_cancelled',
            'order_fraud',
            'service_created',
            'service_suspended',
            'service_terminated',
            'domain_registration',
            'domain_transfer',
            'domain_renewal',
            'domain_expiry',
        ];

        return in_array($notificationName, $available);
    }

    public function getEventIcon(string $event): string
    {
        $icons = [
            'ticket_open' => 'fa-life-ring',
            'ticket_reply' => 'fa-comments',
            'ticket_close' => 'fa-check-circle',
            'invoice_paid' => 'fa-check',
            'order_new' => 'fa-shopping-cart',
            'service_suspended' => 'fa-pause-circle',
            'service_terminated' => 'fa-times-circle',
        ];

        return $icons[$event] ?? 'fa-bell';
    }

    public function getEventArea(): array
    {
        return [
            'admin' => true,
            'client' => true,
        ];
    }
}
```

## Notification Hooks Integration

```php
<?php
// /includes/hooks/notification_hooks.php

// Send custom notification
add_hook('AfterModuleCreate', 1, function(array $params) {
    $notification = new \WHMCS\Module\Notification\YourNotification\Provider();

    $notification->sendNotification(
        [
            'webhookUrl' => get_config('YourNotification_webhookUrl'),
            'apiKey' => get_config('YourNotification_apiKey'),
        ],
        'general',
        'New Service Provisioned',
        "Service #{$params['serviceid']} has been provisioned successfully.",
        [
            'service_id' => $params['serviceid'],
            'domain' => $params['domain'],
            'username' => $params['username'],
            'client' => $params['client']['fullname'],
        ]
    );
});

// Invoice notification
add_hook('InvoicePaid', 1, function(array $params) {
    $notification = new \WHMCS\Module\Notification\YourNotification\Provider();

    $notification->sendNotification(
        [
            'webhookUrl' => get_config('YourNotification_webhookUrl'),
            'template' => 'full',
        ],
        'billing',
        'Invoice Paid',
        "Invoice #{$params['invoiceid']} has been paid.",
        [
            'invoice_id' => $params['invoiceid'],
            'amount' => $params['amount'],
            'payment_method' => $params['paymentmethod'],
        ]
    );
});

// Ticket notifications
add_hook('TicketOpen', 1, function(array $params) {
    $notification = new \WHMCS\Module\Notification\YourNotification\Provider();

    $notification->sendNotification(
        [
            'webhookUrl' => get_config('YourNotification_webhookUrl'),
        ],
        'support',
        'New Support Ticket',
        "Ticket #{$params['ticketid']}: {$params['subject']}",
        [
            'ticket_id' => $params['ticketid'],
            'department' => $params['deptname'],
            'priority' => $params['priority'],
            'client' => $params['client']['fullname'],
        ]
    );
});

// Order notifications
add_hook('OrderPaid', 1, function(array $params) {
    $notification = new \WHMCS\Module\Notification\YourNotification\Provider();

    $status = 'pending';

    if ($params['status'] === 'active') {
        $status = 'completed';
    } elseif ($params['status'] === 'fraud') {
        $status = 'fraud';
    }

    $notification->sendNotification(
        [
            'webhookUrl' => get_config('YourNotification_webhookUrl'),
        ],
        'orders',
        'New Order - ' . ucfirst($status),
        "Order #{$params['orderid']} has status: {$params['status']}",
        [
            'order_id' => $params['orderid'],
            'client' => $params['client']['fullname'],
            'total' => $params['totalamount'],
            'items' => count($params['lineitems']),
        ]
    );
});
```

## Best Practices

1. **Validation**: Always validate configuration before saving
2. **Error Handling**: Wrap webhook calls in try-catch and handle failures gracefully
3. **Rate Limiting**: Implement rate limiting to avoid overwhelming the notification service
4. **Message Formatting**: Support multiple message templates (simple, rich, full)
5. **Color Coding**: Use colors to differentiate notification types
6. **Debug Mode**: Provide debug mode for troubleshooting
7. **Timeout Handling**: Set appropriate timeouts for webhook calls
8. **Fallback**: Provide fallback mechanisms for failed notifications
9. **Security**: Secure API keys and webhook secrets properly
10. **Testing**: Always test with sample notifications before production use
