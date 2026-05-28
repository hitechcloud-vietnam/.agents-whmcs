# WHMCS SMS Gateway Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## Purpose
Create an SMS notification gateway module for WHMCS that allows sending SMS notifications to customers and staff.

## Module Type
Notification Gateway Module

## Use Case
- SMS notifications for orders
- Two-factor authentication (2FA)
- SMS alerts for tickets
- Payment reminders via SMS
- SMS verification codes

## DevKit Structure

```
devkits/whmcs-sms-module/
├── sms.php               # Main SMS gateway module
├── lib/
│   ├── SmsProvider.php    # SMS provider interface
│   ├── Twilio.php        # Twilio implementation
│   ├── Nexmo.php         # Vonage/Nexmo implementation
│   └── MessageBuilder.php # Message composition
├── templates/
│   └── admin.tpl         # Admin templates
├── hooks.php              # Hook integrations
└── DEVKIT.md            # This file
```

## Main Gateway Module Template

```php
<?php
/**
 * WHMCS SMS Gateway Module: {Sms}
 * SMS Notification Gateway Template
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function {sms}_config(): array {
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => '{SMS Gateway}',
        ],
        'description' => [
            'Type' => 'System',
            'Value' => 'SMS notification gateway',
        ],

        // Provider Settings
        'provider' => [
            'FriendlyName' => 'SMS Provider',
            'Type' => 'dropdown',
            'Options' => 'twilio,nexmo,custom',
            'Default' => 'twilio',
        ],

        // Twilio Settings
        'twilio_sid' => [
            'FriendlyName' => 'Account SID (Twilio)',
            'Type' => 'text',
            'Size' => '50',
        ],
        'twilio_token' => [
            'FriendlyName' => 'Auth Token (Twilio)',
            'Type' => 'password',
            'Size' => '50',
        ],
        'twilio_from' => [
            'FriendlyName' => 'From Number (Twilio)',
            'Type' => 'text',
            'Size' => '20',
        ],

        // Nexmo Settings
        'nexmo_key' => [
            'FriendlyName' => 'API Key (Nexmo)',
            'Type' => 'text',
            'Size' => '50',
        ],
        'nexmo_secret' => [
            'FriendlyName' => 'API Secret (Nexmo)',
            'Type' => 'password',
            'Size' => '50',
        ],
        'nexmo_from' => [
            'FriendlyName' => 'From Number (Nexmo)',
            'Type' => 'text',
            'Size' => '20',
        ],

        // Custom Provider Settings
        'custom_api_url' => [
            'FriendlyName' => 'Custom API URL',
            'Type' => 'text',
            'Size' => '80',
        ],
        'custom_api_key' => [
            'FriendlyName' => 'Custom API Key',
            'Type' => 'password',
            'Size' => '50',
        ],

        // General Settings
        'test_mode' => [
            'FriendlyName' => 'Test Mode',
            'Type' => 'yesno',
            'Description' => 'Log messages without sending',
        ],
        'sender_id' => [
            'FriendlyName' => 'Default Sender ID',
            'Type' => 'text',
            'Size' => '20',
            'Description' => 'Default sender name/number',
        ],
        'max_length' => [
            'FriendlyName' => 'Max Message Length',
            'Type' => 'dropdown',
            'Options' => '160,300,500',
            'Default' => '160',
        ],
        'enable_2fa' => [
            'FriendlyName' => 'Enable 2FA via SMS',
            'Type' => 'yesno',
        ],
        'alert_admins' => [
            'FriendlyName' => 'Alert Admins on Failure',
            'Type' => 'yesno',
        ],
        'admin_phone' => [
            'FriendlyName' => 'Admin Phone Number',
            'Type' => 'text',
            'Size' => '20',
        ],
    ];
}

// Send SMS function (used by hooks and API)
function {sms}_send(string $to, string $message, array $options = []): array {
    $settings = getModuleSettings('{sms}');

    // Validate phone number
    $to = formatPhoneNumber($to);

    if (!$to) {
        return ['success' => false, 'error' => 'Invalid phone number'];
    }

    // Check test mode
    if (($settings['test_mode'] ?? '') === 'on') {
        logSmsMessage($to, $message, 'test', 'Test mode - not sent');
        return ['success' => true, 'message_id' => 'test_' . time(), 'test' => true];
    }

    try {
        $provider = getSmsProvider($settings);
        $result = $provider->send($to, $message, $options);

        logSmsMessage($to, $message, 'sent', $result);

        return [
            'success' => true,
            'message_id' => $result['message_id'] ?? time(),
            'segments' => $result['segments'] ?? 1,
        ];
    } catch (\Exception $e) {
        logSmsMessage($to, $message, 'failed', $e->getMessage());

        // Alert admin if enabled
        if (($settings['alert_admins'] ?? '') === 'on') {
            alertAdminSmsFailure($e->getMessage(), $to);
        }

        return ['success' => false, 'error' => $e->getMessage()];
    }
}

// Verify SMS 2FA code
function {sms}_verify_2fa(string $userId, string $code): array {
    $verification = Capsule::table('mod_{sms}_verifications')
        ->where('user_id', $userId)
        ->where('code', $code)
        ->where('expires_at', '>', date('Y-m-d H:i:s'))
        ->where('verified', 0)
        ->first();

    if (!$verification) {
        return ['success' => false, 'error' => 'Invalid or expired code'];
    }

    Capsule::table('mod_{sms}_verifications')
        ->where('id', $verification->id)
        ->update(['verified' => 1]);

    return ['success' => true];
}

// Generate and send 2FA code
function {sms}_send_2fa(int $userId, string $phone): array {
    $code = generateVerificationCode();
    $message = "Your verification code is: {$code}";

    $result = {sms}_send($phone, $message);

    if ($result['success']) {
        Capsule::table('mod_{sms}_verifications')->insert([
            'user_id' => $userId,
            'phone' => $phone,
            'code' => $code,
            'expires_at' => date('Y-m-d H:i:s', strtotime('+10 minutes')),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    return $result;
}

// Activate module
function {sms}_activate(): array {
    try {
        // SMS logs table
        if (!Capsule::schema()->hasTable('mod_{sms}_logs')) {
            Capsule::schema()->create('mod_{sms}_logs', function($t) {
                $t->increments('id');
                $t->string('message_id', 100)->nullable();
                $t->string('to', 20);
                $t->string('from', 20)->nullable();
                $t->text('message');
                $t->string('status', 20);
                $t->string('type', 50)->nullable();
                $t->text('response')->nullable();
                $t->decimal('cost', 10, 4)->nullable();
                $t->timestamp('sent_at');
                $t->timestamps();

                $t->index('to');
                $t->index('status');
                $t->index('sent_at');
            });
        }

        // SMS verifications table (for 2FA)
        if (!Capsule::schema()->hasTable('mod_{sms}_verifications')) {
            Capsule::schema()->create('mod_{sms}_verifications', function($t) {
                $t->increments('id');
                $t->integer('user_id')->unsigned();
                $t->string('phone', 20);
                $t->string('code', 10);
                $t->boolean('verified')->default(false);
                $t->timestamp('expires_at');
                $t->timestamp('created_at');

                $t->index('user_id');
                $t->index('code');
            });
        }

        // SMS templates table
        if (!Capsule::schema()->hasTable('mod_{sms}_templates')) {
            Capsule::schema()->create('mod_{sms}_templates', function($t) {
                $t->increments('id');
                $t->string('name', 100);
                $t->string('event', 100);
                $t->text('message');
                $t->boolean('is_active')->default(true);
                $t->timestamps();

                $t->unique('event');
            });
        }

        // Insert default templates
        $defaultTemplates = [
            ['name' => 'Order Confirmation', 'event' => 'order_created', 'message' => 'Your order #{order_id} has been received. Total: {amount}'],
            ['name' => 'Payment Received', 'event' => 'invoice_paid', 'message' => 'Payment received for invoice #{invoice_id}. Thank you!'],
            ['name' => 'Ticket Reply', 'event' => 'ticket_reply', 'message' => 'New reply to your ticket #{ticket_id}: {subject}'],
            ['name' => '2FA Code', 'event' => '2fa_code', 'message' => 'Your verification code is: {code}'],
        ];

        foreach ($defaultTemplates as $template) {
            Capsule::table('mod_{sms}_templates')->insert($template);
        }

        return ['status' => 'success', 'description' => '{SMS Gateway} activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => 'Activation failed: ' . $e->getMessage()];
    }
}

// Deactivate module
function {sms}_deactivate(): array {
    try {
        Capsule::schema()->dropIfExists('mod_{sms}_logs');
        Capsule::schema()->dropIfExists('mod_{sms}_verifications');
        Capsule::schema()->dropIfExists('mod_{sms}_templates');
        return ['status' => 'success'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => 'Deactivation failed'];
    }
}

// Helper Functions
function getModuleSettings(string $module): array {
    $result = Capsule::table('tbladdon_modules')->where('module', $module)->first();
    return $result ? json_decode($result->value, true) : [];
}

function formatPhoneNumber(string $phone): ?string {
    // Remove all non-digits
    $phone = preg_replace('/[^0-9]/', '', $phone);

    // Add country code if missing (default to US)
    if (strlen($phone) === 10) {
        $phone = '1' . $phone;
    }

    // Validate final format
    if (strlen($phone) >= 10 && strlen($phone) <= 15) {
        return $phone;
    }

    return null;
}

function generateVerificationCode(int $length = 6): string {
    return str_pad((string)random_int(0, pow(10, $length) - 1), $length, '0', STR_PAD_LEFT);
}

function getSmsProvider(array $settings) {
    $provider = $settings['provider'] ?? 'twilio';

    return match ($provider) {
        'twilio' => new \WHMCS\Module\Addon\{Sms}\TwilioProvider($settings),
        'nexmo' => new \WHMCS\Module\Addon\{Sms}\NexmoProvider($settings),
        'custom' => new \WHMCS\Module\Addon\{Sms}\CustomProvider($settings),
        default => new \WHMCS\Module\Addon\{Sms}\TwilioProvider($settings),
    };
}

function logSmsMessage(string $to, string $message, string $status, $response): void {
    Capsule::table('mod_{sms}_logs')->insert([
        'to' => $to,
        'message' => $message,
        'status' => $status,
        'response' => is_string($response) ? $response : json_encode($response),
        'sent_at' => date('Y-m-d H:i:s'),
    ]);
}

function alertAdminSmsFailure(string $error, string $to): void {
    $admins = Capsule::table('tbladmins')
        ->where('disabled', 0)
        ->where('email', '!=', '')
        ->get();

    foreach ($admins as $admin) {
        sendEmail(
            $admin->email,
            'SmsGatewayFailure',
            [
                'error' => $error,
                'recipient' => $to,
            ]
        );
    }
}
```

## SMS Provider Interface

```php
<?php
namespace WHMCS\Module\Addon\{Sms};

interface SmsProviderInterface {
    public function send(string $to, string $message, array $options = []): array;
    public function getBalance(): float;
    public function getMessageStatus(string $messageId): array;
}
```

## Twilio Implementation

```php
<?php
namespace WHMCS\Module\Addon\{Sms};

class TwilioProvider implements SmsProviderInterface {

    private string $accountSid;
    private string $authToken;
    private string $fromNumber;
    private string $baseUrl = 'https://api.twilio.com/2010-04-01';

    public function __construct(array $settings) {
        $this->accountSid = $settings['twilio_sid'] ?? '';
        $this->authToken = $settings['twilio_token'] ?? '';
        $this->fromNumber = $settings['twilio_from'] ?? '';
    }

    public function send(string $to, string $message, array $options = []): array {
        $url = "{$this->baseUrl}/Accounts/{$this->accountSid}/Messages.json";

        $data = [
            'To' => '+' . $to,
            'From' => $options['from'] ?? $this->fromNumber,
            'Body' => $message,
        ];

        if (isset($options['status_callback'])) {
            $data['StatusCallback'] = $options['status_callback'];
        }

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($data),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_USERPWD => "{$this->accountSid}:{$this->authToken}",
            CURLOPT_TIMEOUT => 30,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception('cURL Error: ' . $error);
        }

        $result = json_decode($response, true);

        if ($httpCode >= 400) {
            throw new \Exception($result['message'] ?? 'Twilio API Error');
        }

        return [
            'message_id' => $result['sid'] ?? '',
            'status' => $result['status'] ?? 'queued',
            'segments' => $result['num_segments'] ?? 1,
            'price' => $result['price'] ?? 0,
        ];
    }

    public function getBalance(): float {
        $url = "{$this->baseUrl}/Accounts/{$this->accountSid}.json";

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_USERPWD => "{$this->accountSid}:{$this->authToken}",
            CURLOPT_TIMEOUT => 30,
        ]);

        $response = curl_exec($ch);
        $result = json_decode($response, true);
        curl_close($ch);

        return (float)($result['balance'] ?? 0);
    }

    public function getMessageStatus(string $messageId): array {
        $url = "{$this->baseUrl}/Accounts/{$this->accountSid}/Messages/{$messageId}.json";

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_USERPWD => "{$this->accountSid}:{$this->authToken}",
            CURLOPT_TIMEOUT => 30,
        ]);

        $response = curl_exec($ch);
        $result = json_decode($response, true);
        curl_close($ch);

        return [
            'status' => $result['status'] ?? 'unknown',
            'error_code' => $result['error_code'] ?? null,
            'error_message' => $result['error_message'] ?? null,
        ];
    }
}
```

## Nexmo Implementation

```php
<?php
namespace WHMCS\Module\Addon\{Sms};

class NexmoProvider implements SmsProviderInterface {

    private string $apiKey;
    private string $apiSecret;
    private string $fromNumber;
    private string $baseUrl = 'https://rest.nexmo.com';

    public function __construct(array $settings) {
        $this->apiKey = $settings['nexmo_key'] ?? '';
        $this->apiSecret = $settings['nexmo_secret'] ?? '';
        $this->fromNumber = $settings['nexmo_from'] ?? '';
    }

    public function send(string $to, string $message, array $options = []): array {
        $url = "{$this->baseUrl}/sms/json";

        $data = [
            'api_key' => $this->apiKey,
            'api_secret' => $this->apiSecret,
            'to' => $to,
            'from' => $options['from'] ?? $this->fromNumber,
            'text' => $message,
        ];

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($data),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception('cURL Error: ' . $error);
        }

        $result = json_decode($response, true);
        $messageResult = $result['messages'][0] ?? [];

        if ($messageResult['status'] !== '0') {
            throw new \Exception($messageResult['error-text'] ?? 'Nexmo API Error');
        }

        return [
            'message_id' => $messageResult['message-id'] ?? '',
            'status' => 'delivered',
            'segments' => $messageResult['messages-count'] ?? 1,
            'price' => $messageResult['message-price'] ?? 0,
        ];
    }

    public function getBalance(): float {
        $url = "{$this->baseUrl}/account/get-balance?api_key={$this->apiKey}&api_secret={$this->apiSecret}";

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);

        $response = curl_exec($ch);
        $result = json_decode($response, true);
        curl_close($ch);

        return (float)($result['value'] ?? 0);
    }

    public function getMessageStatus(string $messageId): array {
        return ['status' => 'unknown'];
    }
}
```

## Custom Provider Implementation

```php
<?php
namespace WHMCS\Module\Addon\{Sms};

class CustomProvider implements SmsProviderInterface {

    private string $apiUrl;
    private string $apiKey;
    private string $fromNumber;

    public function __construct(array $settings) {
        $this->apiUrl = $settings['custom_api_url'] ?? '';
        $this->apiKey = $settings['custom_api_key'] ?? '';
        $this->fromNumber = $settings['sender_id'] ?? '';
    }

    public function send(string $to, string $message, array $options = []): array {
        $data = [
            'to' => $to,
            'from' => $options['from'] ?? $this->fromNumber,
            'message' => $message,
        ];

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->apiUrl,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'Authorization: Bearer ' . $this->apiKey,
            ],
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception('cURL Error: ' . $error);
        }

        $result = json_decode($response, true);

        if ($httpCode >= 400) {
            throw new \Exception($result['error'] ?? 'API Error');
        }

        return [
            'message_id' => $result['id'] ?? $result['message_id'] ?? time(),
            'status' => $result['status'] ?? 'sent',
            'segments' => 1,
        ];
    }

    public function getBalance(): float {
        return 0;
    }

    public function getMessageStatus(string $messageId): array {
        return ['status' => 'unknown'];
    }
}
```

## Hooks Integration

```php
<?php
/**
 * WHMCS SMS Module Hooks
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// Send SMS on order creation
add_hook('AcceptOrder', 1, function(array $vars) {
    $orderId = $vars['orderid'];
    $order = Capsule::table('tblorders')->where('id', $orderId)->first();

    if (!$order) {
        return;
    }

    $client = Capsule::table('tblclients')->where('id', $order->userid)->first();

    if (!$client || empty($client->phonenumber)) {
        return;
    }

    $template = Capsule::table('mod_{sms}_templates')
        ->where('event', 'order_created')
        ->where('is_active', 1)
        ->first();

    if (!$template) {
        return;
    }

    $message = str_replace(
        ['{order_id}', '{amount}'],
        [$orderId, $order->amount],
        $template->message
    );

    {sms}_send($client->phonenumber, $message, ['type' => 'order']);
});

// Send SMS on invoice payment
add_hook('InvoicePaid', 1, function(array $vars) {
    $invoiceId = $vars['invoiceid'];
    $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();

    if (!$invoice) {
        return;
    }

    $client = Capsule::table('tblclients')->where('id', $invoice->userid)->first();

    if (!$client || empty($client->phonenumber)) {
        return;
    }

    $template = Capsule::table('mod_{sms}_templates')
        ->where('event', 'invoice_paid')
        ->where('is_active', 1)
        ->first();

    if (!$template) {
        return;
    }

    $message = str_replace(
        ['{invoice_id}', '{amount}'],
        [$invoiceId, $invoice->total],
        $template->message
    );

    {sms}_send($client->phonenumber, $message, ['type' => 'payment']);
});

// Send SMS on ticket reply
add_hook('TicketReply', 1, function(array $vars) {
    $ticketId = $vars['ticketid'];
    $ticket = Capsule::table('tbltickets')->where('id', $ticketId)->first();

    if (!$ticket) {
        return;
    }

    $client = Capsule::table('tblclients')->where('id', $ticket->userid)->first();

    if (!$client || empty($client->phonenumber)) {
        return;
    }

    $template = Capsule::table('mod_{sms}_templates')
        ->where('event', 'ticket_reply')
        ->where('is_active', 1)
        ->first();

    if (!$template) {
        return;
    }

    $message = str_replace(
        ['{ticket_id}', '{subject}'],
        [$ticket->tid, substr($ticket->subject, 0, 50)],
        $template->message
    );

    {sms}_send($client->phonenumber, $message, ['type' => 'ticket']);
});
```

## Database Schema

### mod_{sms}_logs
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| message_id | VARCHAR(100) | Provider message ID |
| to | VARCHAR(20) | Recipient phone |
| from | VARCHAR(20) | Sender |
| message | TEXT | Message content |
| status | VARCHAR(20) | sent/failed/test |
| type | VARCHAR(50) | order/payment/ticket |
| response | TEXT | API response |
| cost | DECIMAL(10,4) | Message cost |
| sent_at | TIMESTAMP | Send time |

### mod_{sms}_verifications
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| user_id | INT | User ID |
| phone | VARCHAR(20) | Phone number |
| code | VARCHAR(10) | Verification code |
| verified | BOOLEAN | Verified status |
| expires_at | TIMESTAMP | Expiration time |

## Checklist

```
Pre-Dev:
□ Choose SMS provider (Twilio, Nexmo, etc.)
□ Get API credentials
□ Plan message templates
□ Design 2FA flow

Development:
□ Implement config() with all settings
□ Implement activate() → Create tables
□ Implement deactivate() → Drop tables
□ Create SmsProvider interface
□ Create TwilioProvider implementation
□ Create NexmoProvider implementation
□ Create CustomProvider implementation
□ Implement send() function
□ Implement 2FA functions
□ Add hooks for notifications
□ Create message templates

Security:
□ Validate phone numbers
□ Secure API credentials
□ Implement rate limiting
□ Log all SMS sends
□ Handle API failures

Testing:
□ Test SMS sending
□ Test with sandbox credentials
□ Verify message formatting
□ Test 2FA flow
□ Verify hook integration
□ Test error handling
```
