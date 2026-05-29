# WHMCS Mail Provider Master

## Overview
Master skill for WHMCS custom mail provider development. Covers SMTP configuration, API-based email delivery, template rendering, and email tracking.

## Mail Provider Structure

```php
<?php
// /modules/mail/YourMailProvider/YourMailProvider.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Mail\Message;
use WHMCS\Exception\Mail\SendFailure;

function YourMailProvider_Settings()
{
    return [
        'name' => [
            'FriendlyName' => 'Provider Name',
            'Type' => 'system',
            'Description' => 'Your Mail Provider',
        ],
        'apiKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Required' => true,
            'Description' => 'Your mail provider API key',
        ],
        'apiSecret' => [
            'FriendlyName' => 'API Secret',
            'Type' => 'password',
            'Required' => false,
            'Description' => 'Your mail provider API secret (if required)',
        ],
        'fromEmail' => [
            'FriendlyName' => 'Default From Email',
            'Type' => 'text',
            'Required' => true,
            'Description' => 'Default sender email address',
        ],
        'fromName' => [
            'FriendlyName' => 'Default From Name',
            'Type' => 'text',
            'Required' => false,
            'Description' => 'Default sender name',
        ],
        'replyTo' => [
            'FriendlyName' => 'Reply-To Address',
            'Type' => 'text',
            'Required' => false,
            'Description' => 'Default reply-to address',
        ],
        'template' => [
            'FriendlyName' => 'Email Template',
            'Type' => 'dropdown',
            'Options' => [
                'html' => 'HTML Template',
                'markdown' => 'Markdown Template',
                'raw' => 'Raw HTML',
            ],
            'Default' => 'html',
            'Description' => 'Email content template type',
        ],
        'trackOpens' => [
            'FriendlyName' => 'Track Opens',
            'Type' => 'yesno',
            'Description' => 'Enable open tracking',
        ],
        'trackClicks' => [
            'FriendlyName' => 'Track Clicks',
            'Type' => 'yesno',
            'Description' => 'Enable click tracking',
        ],
        'webhookUrl' => [
            'FriendlyName' => 'Webhook URL',
            'Type' => 'text',
            'Required' => false,
            'Description' => 'Webhook for delivery and bounce notifications',
        ],
        'sandbox' => [
            'FriendlyName' => 'Sandbox Mode',
            'Type' => 'yesno',
            'Description' => 'Send to sandbox only (testing)',
        ];
}

function YourMailProvider_Send(array $params)
{
    try {
        $api = new MailProviderApi($params['settings']);

        $message = [
            'to' => formatRecipients($params['recipients']),
            'from' => [
                'email' => $params['senderEmail'] ?: $params['settings']['fromEmail'],
                'name' => $params['senderName'] ?: $params['settings']['fromName'],
            ],
            'subject' => $params['subject'],
            'html' => $params['body'],
            'text' => strip_tags($params['body']),
            'headers' => [
                'X-Mailer' => 'WHMCS',
                'X-MessageId' => $params['id'],
            ],
        ];

        if (!empty($params['replyTo'])) {
            $message['reply_to'] = formatRecipients([$params['replyTo']]);
        }

        if (!empty($params['attachments'])) {
            $message['attachments'] = formatAttachments($params['attachments']);
        }

        // Add tracking if enabled
        if (!empty($params['settings']['trackOpens'])) {
            $message['tracking'] = [
                'opens' => true,
            ];
        }

        if (!empty($params['settings']['trackClicks'])) {
            $message['tracking'] = [
                'clicks' => true,
            ];
        }

        // Add custom headers
        if (!empty($params['customHeader'])) {
            foreach ($params['customHeader'] as $key => $value) {
                $message['headers'][$key] = $value;
            }
        }

        $result = $api->send($message);

        if ($result['success']) {
            return [
                'result' => 'success',
                'messageId' => $result['message_id'],
            ];
        }

        throw new SendFailure($result['error'] ?? 'Failed to send email');

    } catch (\Exception $e) {
        return [
            'result' => 'error',
            'error' => $e->getMessage(),
        ];
    }
}

function YourMailProvider_TestConnection(array $params)
{
    try {
        $api = new MailProviderApi($params['settings']);

        $result = $api->verify();

        return [
            'success' => $result['success'],
            'error' => $result['error'] ?? null,
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

function YourMailProvider_GetStats(array $params)
{
    try {
        $api = new MailProviderApi($params['settings']);

        $result = $api->getStats([
            'start_date' => date('Y-m-01'),
            'end_date' => date('Y-m-d'),
        ]);

        return [
            'success' => true,
            'stats' => [
                'sent' => $result['sent'],
                'delivered' => $result['delivered'],
                'bounced' => $result['bounced'],
                'opened' => $result['opened'],
                'clicked' => $result['clicked'],
                'unsubscribed' => $result['unsubscribed'],
            ],
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

function formatRecipients(array $recipients): array
{
    $formatted = [];

    foreach ($recipients as $recipient) {
        if (is_array($recipient)) {
            $formatted[] = [
                'email' => $recipient['email'],
                'name' => $recipient['name'] ?? null,
            ];
        } else {
            $formatted[] = [
                'email' => $recipient,
            ];
        }
    }

    return $formatted;
}

function formatAttachments(array $attachments): array
{
    $formatted = [];

    foreach ($attachments as $attachment) {
        if (is_array($attachment)) {
            $formatted[] = [
                'content' => base64_encode($attachment['data']),
                'filename' => $attachment['name'],
                'type' => $attachment['type'],
            ];
        } else {
            $formatted[] = [
                'content' => base64_encode(file_get_contents($attachment)),
                'filename' => basename($attachment),
            ];
        }
    }

    return $formatted;
}
```

## Mail Provider API Client

```php
<?php
// /modules/mail/YourMailProvider/lib/MailProviderApi.php

namespace WHMCS\Mail\YourMailProvider;

class MailProviderApi
{
    private $apiKey;
    private $apiSecret;
    private $sandbox;
    private $webhookUrl;

    const API_URL = 'https://api.mailprovider.com/v1';
    const SANDBOX_URL = 'https://sandbox.mailprovider.com/v1';

    public function __construct(array $settings)
    {
        $this->apiKey = $settings['apiKey'] ?? '';
        $this->apiSecret = $settings['apiSecret'] ?? '';
        $this->sandbox = $settings['sandbox'] ?? false;
        $this->webhookUrl = $settings['webhookUrl'] ?? '';
    }

    private function getBaseUrl(): string
    {
        return $this->sandbox ? self::SANDBOX_URL : self::API_URL;
    }

    public function send(array $message): array
    {
        $url = $this->getBaseUrl() . '/messages/send';

        $payload = $this->preparePayload($message);

        $response = $this->makeRequest('POST', $url, $payload);

        return [
            'success' => isset($response['id']),
            'message_id' => $response['id'] ?? null,
            'error' => $response['error'] ?? null,
        ];
    }

    public function sendBulk(array $messages): array
    {
        $url = $this->getBaseUrl() . '/messages/send/batch';

        $payload = [];
        foreach ($messages as $message) {
            $payload[] = $this->preparePayload($message);
        }

        $response = $this->makeRequest('POST', $url, ['batch' => $payload]);

        return [
            'success' => true,
            'results' => $response['results'] ?? [],
        ];
    }

    public function verify(): array
    {
        $url = $this->getBaseUrl() . '/account/verify';

        try {
            $response = $this->makeRequest('GET', $url);

            return [
                'success' => !empty($response['account_id']),
                'account' => $response,
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage(),
            ];
        }
    }

    public function getStats(array $params = []): array
    {
        $url = $this->getBaseUrl() . '/stats?' . http_build_query($params);

        $response = $this->makeRequest('GET', $url);

        return [
            'sent' => $response['sent'] ?? 0,
            'delivered' => $response['delivered'] ?? 0,
            'bounced' => $response['bounced'] ?? 0,
            'opened' => $response['opened'] ?? 0,
            'clicked' => $response['clicked'] ?? 0,
            'unsubscribed' => $response['unsubscribed'] ?? 0,
        ];
    }

    public function getMessageStatus(string $messageId): array
    {
        $url = $this->getBaseUrl() . '/messages/' . $messageId;

        return $this->makeRequest('GET', $url);
    }

    public function getBounces(array $params = []): array
    {
        $url = $this->getBaseUrl() . '/bounces?' . http_build_query($params);

        return $this->makeRequest('GET', $url);
    }

    public function getSpamReports(array $params = []): array
    {
        $url = $this->getBaseUrl() . '/spam_reports?' . http_build_query($params);

        return $this->makeRequest('GET', $url);
    }

    public function createTemplate(string $name, string $subject, string $html, string $text = ''): array
    {
        $url = $this->getBaseUrl() . '/templates';

        return $this->makeRequest('POST', $url, [
            'name' => $name,
            'subject' => $subject,
            'html' => $html,
            'text' => $text,
        ]);
    }

    public function createList(string $name, string $description = ''): array
    {
        $url = $this->getBaseUrl() . '/lists';

        return $this->makeRequest('POST', $url, [
            'name' => $name,
            'description' => $description,
        ]);
    }

    public function addListMember(string $listId, string $email, array $mergeFields = []): array
    {
        $url = $this->getBaseUrl() . '/lists/' . $listId . '/members';

        return $this->makeRequest('POST', $url, array_merge([
            'email_address' => $email,
            'status' => 'subscribed',
        ], $mergeFields));
    }

    private function preparePayload(array $message): array
    {
        $payload = [
            'from' => $message['from'],
            'to' => $message['to'],
            'subject' => $message['subject'],
        ];

        if (!empty($message['html'])) {
            $payload['html'] = $message['html'];
        }

        if (!empty($message['text'])) {
            $payload['text'] = $message['text'];
        }

        if (!empty($message['reply_to'])) {
            $payload['reply_to'] = $message['reply_to'][0] ?? $message['reply_to'];
        }

        if (!empty($message['attachments'])) {
            $payload['attachments'] = $message['attachments'];
        }

        if (!empty($message['headers'])) {
            $payload['headers'] = $message['headers'];
        }

        if (!empty($message['tracking'])) {
            $payload['tracking_settings'] = $message['tracking'];
        }

        if (!empty($message['custom_args'])) {
            $payload['custom_args'] = $message['custom_args'];
        }

        if ($this->webhookUrl) {
            $payload['webhook_url'] = $this->webhookUrl;
        }

        return $payload;
    }

    private function makeRequest(string $method, string $url, array $data = []): array
    {
        $headers = [
            'Authorization: Bearer ' . $this->apiKey,
            'Content-Type: application/json',
        ];

        $ch = curl_init();

        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 60,
            CURLOPT_HTTPHEADER => $headers,
        ]);

        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception('cURL Error: ' . $error);
        }

        $result = json_decode($response, true);

        if ($httpCode >= 400) {
            $errorMessage = $result['error']['message'] ?? 'API Error';
            throw new \Exception($errorMessage);
        }

        return $result;
    }
}
```

## Webhook Handler for Email Events

```php
<?php
// /modules/mail/YourMailProvider/webhook.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/MailProviderApi.php';

// Verify webhook signature
$payload = file_get_contents('php://input');
$signature = $_SERVER['HTTP_X_SIGNATURE'] ?? '';

$settings = require __DIR__ . '/settings.php';

if (!verifySignature($payload, $signature, $settings['webhookSecret'])) {
    http_response_code(401);
    die('Unauthorized');
}

$event = json_decode($payload, true);
$eventType = $event['event'] ?? '';
$messageId = $event['message_id'] ?? '';

logTransaction('YourMailProvider', $event, 'Webhook: ' . $eventType);

switch ($eventType) {
    case 'delivered':
        handleDelivered($event);
        break;

    case 'bounce':
        handleBounce($event);
        break;

    case 'dropped':
        handleDropped($event);
        break;

    case 'open':
        handleOpen($event);
        break;

    case 'click':
        handleClick($event);
        break;

    case 'unsubscribe':
        handleUnsubscribe($event);
        break;

    case 'spam_report':
        handleSpamReport($event);
        break;
}

http_response_code(200);
echo 'OK';

function handleDelivered(array $event): void
{
    // Update delivery status in WHMCS
    if (!empty($event['message_id'])) {
        \Illuminate\Database\Capsule\Manager::table('tblemails')
            ->where('id', $event['message_id'])
            ->update([
                'delivery_status' => 'delivered',
                'delivered_at' => date('Y-m-d H:i:s'),
            ]);
    }
}

function handleBounce(array $event): void
{
    $email = $event['email'] ?? '';
    $bounceType = $event['bounce_type'] ?? 'unknown';
    $reason = $event['reason'] ?? '';

    // Update email record
    if (!empty($event['message_id'])) {
        \Illuminate\Database\Capsule\Manager::table('tblemails')
            ->where('id', $event['message_id'])
            ->update([
                'delivery_status' => 'bounced',
                'bounce_reason' => $reason,
            ]);
    }

    // Mark client email as bounced if critical
    if ($bounceType === 'hard_bounce') {
        $client = \WHMCS\User\Client::where('email', $email)->first();
        if ($client) {
            $client->email = $client->email . '.bounced';
            $client->save();
        }
    }
}

function handleDropped(array $event): void
{
    $reason = $event['reason'] ?? '';

    if (!empty($event['message_id'])) {
        \Illuminate\Database\Capsule\Manager::table('tblemails')
            ->where('id', $event['message_id'])
            ->update([
                'delivery_status' => 'dropped',
                'drop_reason' => $reason,
            ]);
    }
}

function handleOpen(array $event): void
{
    // Log email open event
    if (!empty($event['message_id'])) {
        logActivity("[Email Open] Message ID: " . $event['message_id']);
    }
}

function handleClick(array $event): void
{
    // Log email click event
    if (!empty($event['message_id'])) {
        logActivity("[Email Click] Message ID: " . $event['message_id'] . " - URL: " . $event['url']);
    }
}

function handleUnsubscribe(array $event): void
{
    $email = $event['email'] ?? '';

    // Add to unsubscribe list in WHMCS
    // This would require custom implementation
}

function handleSpamReport(array $event): void
{
    $email = $event['email'] ?? '';

    logActivity("[Spam Report] Email: " . $email);

    // Could flag the client account
}

function verifySignature(string $payload, string $signature, string $secret): bool
{
    $expectedSignature = hash_hmac('sha256', $payload, $secret);
    return hash_equals($expectedSignature, $signature);
}
```

## Best Practices

1. **API Security**: Use API keys and signature verification for webhooks
2. **Error Handling**: Implement proper error handling with fallback to local mail
3. **Rate Limiting**: Respect API rate limits and implement queuing
4. **Tracking**: Enable open and click tracking for analytics
5. **Bounce Handling**: Process bounces and update email status
6. **Template Management**: Support email template management
7. **Attachment Support**: Handle file attachments properly
8. **SMTP Fallback**: Provide SMTP fallback for failed sends
9. **Logging**: Log all email sends for auditing
10. **Testing**: Test with sandbox mode before production
