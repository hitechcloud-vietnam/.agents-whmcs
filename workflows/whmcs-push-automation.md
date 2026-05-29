# WHMCS Push Automation Workflow

## Overview
This workflow implements push notification automation for WHMCS client applications.

## Prerequisites
- WHMCS with mobile app integration
- Push notification service (Firebase, OneSignal, etc.)
- Mobile app development capability

## Step-by-Step Process

### Step 1: Create Push Notification Manager
```php
<?php
// /includes/push/PushNotificationManager.php

namespace WHMCS\Push;

class PushNotificationManager
{
    private $provider;
    private $enabled = false;

    public function __construct()
    {
        $this->loadProvider();
    }

    private function loadProvider()
    {
        $providerType = getConfig('push_provider');

        switch ($providerType) {
            case 'fcm':
                $this->provider = new FirebaseCloudMessagingProvider();
                $this->enabled = true;
                break;
            case 'onesignal':
                $this->provider = new OneSignalProvider();
                $this->enabled = true;
                break;
            default:
                $this->enabled = false;
        }
    }

    /**
     * Send push notification to user
     */
    public function send(int $clientId, string $template, array $data = [], array $options = []): array
    {
        if (!$this->enabled) {
            return ['success' => false, 'error' => 'Push notifications disabled'];
        }

        // Get user's device tokens
        $tokens = $this->getDeviceTokens($clientId);

        if (empty($tokens)) {
            return ['success' => false, 'error' => 'No device tokens'];
        }

        // Render notification content
        $notification = $this->renderNotification($template, $data, $clientId);

        // Send to all user's devices
        $results = [];
        foreach ($tokens as $token) {
            $results[] = $this->provider->send($token, $notification, $options);
        }

        // Log notification
        $this->logNotification($clientId, $template, $notification, $results);

        return [
            'success' => true,
            'sent' => count(array_filter($results, fn($r) => $r['success'])),
            'failed' => count(array_filter($results, fn($r) => !$r['success']))
        ];
    }

    /**
     * Register device token
     */
    public function registerDevice(int $clientId, string $token, string $platform): bool
    {
        // Check if token exists
        $existing = Capsule::table('mod_push_tokens')
            ->where('token', $token)
            ->first();

        if ($existing) {
            // Update client association
            Capsule::table('mod_push_tokens')
                ->where('token', $token)
                ->update([
                    'client_id' => $clientId,
                    'platform' => $platform,
                    'updated_at' => date('Y-m-d H:i:s')
                ]);
        } else {
            // Insert new token
            Capsule::table('mod_push_tokens')->insert([
                'client_id' => $clientId,
                'token' => $token,
                'platform' => $platform,
                'created_at' => date('Y-m-d H:i:s')
            ]);
        }

        return true;
    }

    /**
     * Remove device token
     */
    public function removeDevice(string $token): bool
    {
        return Capsule::table('mod_push_tokens')
            ->where('token', $token)
            ->delete() > 0;
    }

    private function getDeviceTokens(int $clientId): array
    {
        return Capsule::table('mod_push_tokens')
            ->where('client_id', $clientId)
            ->where('enabled', 1)
            ->pluck('token')
            ->toArray();
    }

    private function renderNotification(string $template, array $data, int $clientId): array
    {
        $client = getClientsDetails($clientId);

        $templates = [
            'payment_received' => [
                'title' => 'Payment Received',
                'body' => "Hi {$client['firstname']}, we received your payment of {$data['amount']}",
                'data' => ['type' => 'payment', 'invoice_id' => $data['invoice_id'] ?? null]
            ],
            'service_activated' => [
                'title' => 'Service Activated',
                'body' => "Hi {$client['firstname']}, your service is now ready!",
                'data' => ['type' => 'activation', 'service_id' => $data['service_id'] ?? null]
            ],
            'service_suspended' => [
                'title' => 'Service Suspended',
                'body' => "Hi {$client['firstname']}, your service has been suspended",
                'data' => ['type' => 'suspension', 'service_id' => $data['service_id'] ?? null]
            ],
            'new_ticket_reply' => [
                'title' => 'New Ticket Reply',
                'body' => "You have a new reply on ticket #{$data['ticket_id']}",
                'data' => ['type' => 'support', 'ticket_id' => $data['ticket_id'] ?? null]
            ],
            'renewal_reminder' => [
                'title' => 'Renewal Reminder',
                'body' => "Your service renews on {$data['due_date']}",
                'data' => ['type' => 'renewal', 'service_id' => $data['service_id'] ?? null]
            ]
        ];

        return $templates[$template] ?? [
            'title' => 'Notification',
            'body' => $template,
            'data' => $data
        ];
    }

    private function logNotification(int $clientId, string $template, array $notification, array $results)
    {
        Capsule::table('mod_push_log')->insert([
            'client_id' => $clientId,
            'template' => $template,
            'title' => $notification['title'],
            'body' => $notification['body'],
            'success_count' => count(array_filter($results, fn($r) => $r['success'])),
            'failure_count' => count(array_filter($results, fn($r) => !$r['success'])),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

### Step 2: Create Push Provider Implementation
```php
<?php
// /includes/push/providers/FirebaseCloudMessagingProvider.php

class FirebaseCloudMessagingProvider
{
    private $serverKey;
    private $projectId;

    public function __construct()
    {
        $this->serverKey = getConfig('fcm_server_key');
        $this->projectId = getConfig('fcm_project_id');
    }

    public function send(string $token, array $notification, array $options = []): array
    {
        $url = "https://fcm.googleapis.com/fcm/send";

        $payload = [
            'to' => $token,
            'notification' => [
                'title' => $notification['title'],
                'body' => $notification['body'],
                'sound' => $options['sound'] ?? 'default',
                'badge' => $options['badge'] ?? 1
            ],
            'data' => $notification['data'] ?? []
        ];

        // Add Android-specific options
        if (isset($options['android'])) {
            $payload['android'] = $options['android'];
        }

        // Add iOS-specific options
        if (isset($options['ios'])) {
            $payload['apns'] = [
                'payload' => [
                    'aps' => $options['ios']
                ]
            ];
        }

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_HTTPHEADER => [
                'Authorization: key=' . $this->serverKey,
                'Content-Type: application/json'
            ],
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $result = json_decode($response, true);

        if ($httpCode === 200 && isset($result['success']) && $result['success'] === 1) {
            return [
                'success' => true,
                'message_id' => $result['results'][0]['message_id'] ?? null
            ];
        }

        return [
            'success' => false,
            'error' => $result['results'][0]['error'] ?? 'Unknown error'
        ];
    }

    public function sendBatch(array $tokens, array $notification): array
    {
        $url = "https://fcm.googleapis.com/fcm/send";

        $payload = [
            'registration_ids' => $tokens,
            'notification' => $notification,
            'apns' => [
                'payload' => ['aps' => ['content-available' => 1]]
            ]
        ];

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_HTTPHEADER => [
                'Authorization: key=' . $this->serverKey,
                'Content-Type: application/json'
            ],
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        $result = json_decode($response, true);

        return [
            'success' => $result['success'] ?? 0,
            'failure' => $result['failure'] ?? 0
        ];
    }
}
```

### Step 3: Create Push Hooks
```php
<?php
// /includes/hooks/push_hooks.php

use WHMCS\Push\PushNotificationManager;

$pushManager = new PushNotificationManager();

// New payment
add_hook('InvoicePaid', 1, function($vars) use ($pushManager) {
    $invoice = getInvoice($vars['invoiceid']);

    if (clientHasPushEnabled($invoice['userid'])) {
        $pushManager->send($invoice['userid'], 'payment_received', [
            'amount' => formatCurrency($invoice['total'])
        ]);
    }
});

// Service status changes
add_hook('ServiceSuspended', 1, function($vars) use ($pushManager) {
    if (clientHasPushEnabled($vars['userid'])) {
        $pushManager->send($vars['userid'], 'service_suspended', [
            'service_id' => $vars['serviceid']
        ], ['priority' => 'high']);
    }
});

add_hook('ServiceUnsuspended', 1, function($vars) use ($pushManager) {
    if (clientHasPushEnabled($vars['userid'])) {
        $pushManager->send($vars['userid'], 'service_restored', [
            'service_id' => $vars['serviceid']
        ]);
    }
});

// Ticket replies
add_hook('TicketReply', 1, function($vars) use ($pushManager) {
    $ticket = Capsule::table('tbltickets')
        ->where('id', $vars['ticketid'])
        ->first();

    if (clientHasPushEnabled($ticket->userid)) {
        $pushManager->send($ticket->userid, 'new_ticket_reply', [
            'ticket_id' => $vars['ticketid'],
            'message' => substr($vars['message'], 0, 100)
        ]);
    }
});
```

## Push Notification Best Practices

1. **Respect user preferences** - Honor quiet hours and opt-outs
2. **Keep notifications concise** - Brief, actionable messages
3. **Use rich notifications** - Include images and actions
4. **Personalize content** - Use client's name and relevant data
5. **Test across platforms** - iOS and Android may differ
6. **Monitor delivery** - Track success rates

## Related Workflows
- [WHMCS Notification Automation](./whmcs-notification-automation.md)
- [WHMCS SMS Automation](./whmcs-sms-automation.md)
- [WHMCS Email Automation](./whmcs-email-automation.md)