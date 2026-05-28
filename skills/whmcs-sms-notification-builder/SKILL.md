# WHMCS SMS Notification Builder Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building SMS notification providers and integrations.

## When to Use

- Creating SMS gateway modules
- Building multi-channel notifications
- Implementing OTP/2FA via SMS

## SMS Notification Patterns

```php
<?php
// modules/notifications/{smsprovider}/{smsprovider}.php

namespace WHMCS\Module\Notification\{SmsProvider};

use WHMCS\Module\Contracts\NotificationModuleInterface;
use WHMCS\Notification\Contracts\NotificationInterface;
use WHMCS\Module\Notification\DescriptionTrait;

class Provider implements NotificationModuleInterface {
    use DescriptionTrait;

    public static function moduleConfiguration(): array {
        return [
            ['Name' => 'api_key', 'Type' => 'password', 'FriendlyName' => 'API Key'],
            ['Name' => 'api_secret', 'Type' => 'password', 'FriendlyName' => 'API Secret'],
            ['Name' => 'sender_name', 'Type' => 'text', 'FriendlyName' => 'Sender Name'],
            ['Name' => 'test_mode', 'Type' => 'yesno', 'FriendlyName' => 'Test Mode'],
        ];
    }

    public function notificationSettings(): array {
        return [
            ['Name' => 'phone_field', 'Type' => 'dropdown', 'FriendlyName' => 'Phone Field',
             'Options' => 'phone,mobilephone,custom1,custom2'],
            ['Name' => 'template', 'Type' => 'text', 'FriendlyName' => 'SMS Template ID'],
        ];
    }

    public function testConnection(): void {
        $api = new SMS\ApiClient($this->getSettings());
        $result = $api->sendSMS('test', 'This is a test message');
        if (!$result) {
            throw new \Exception('Could not connect to SMS API');
        }
    }

    public function send(NotificationInterface $notification, array $settings): void {
        $api = new SMS\ApiClient($settings);

        $phone = $this->getPhoneNumber($notification, $settings);
        $message = $this->formatMessage($notification);

        $result = $api->sendSMS($phone, $message);

        if (!$result['success']) {
            throw new \Exception('Failed to send SMS: ' . ($result['error'] ?? 'Unknown error'));
        }

        // Log the SMS
        $this->logSMS($notification, $phone, $message, $result);
    }

    private function getPhoneNumber(NotificationInterface $notification, array $settings): string {
        $client = $notification->getClient();
        $field = $settings['phone_field'] ?? 'mobilephone';

        return $client->$field ?? '';
    }

    private function formatMessage(NotificationInterface $notification): string {
        $title = $notification->getTitle();
        $message = $notification->getMessage();
        $url = $notification->getUrl();

        $text = $title;
        if ($message) {
            $text .= "\n" . $message;
        }
        if ($url) {
            $text .= "\n" . $url;
        }

        // SMS max length
        return mb_substr($text, 0, 160);
    }

    private function logSMS(NotificationInterface $notification, string $phone, string $message, array $result): void {
        Capsule::table('mod_sms_logs')->insert([
            'notification_type' => get_class($notification),
            'phone' => $phone,
            'message' => $message,
            'status' => $result['success'] ? 'sent' : 'failed',
            'api_message_id' => $result['message_id'] ?? null,
            'sent_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

### OTP/2FA SMS Pattern

```php
<?php
// modules/addons/{smsmodule}/lib/OTPService.php

namespace {Module};

class OTPService {
    private int $length = 6;
    private int $expiryMinutes = 5;

    public function generateOTP(string $phone): string {
        $otp = '';
        for ($i = 0; $i < $this->length; $i++) {
            $otp .= random_int(0, 9);
        }

        // Store in database with expiry
        Capsule::table('mod_sms_otp')->insert([
            'phone' => $phone,
            'otp' => hash('sha256', $otp),
            'attempts' => 0,
            'expires_at' => date('Y-m-d H:i:s', strtotime("+{$this->expiryMinutes} minutes")),
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        // Send SMS
        $this->sendOTP($phone, $otp);

        return $otp;
    }

    public function verifyOTP(string $phone, string $enteredOtp): bool {
        $record = Capsule::table('mod_sms_otp')
            ->where('phone', $phone)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->where('attempts', '<', 5)
            ->orderBy('created_at', 'desc')
            ->first();

        if (!$record) {
            return false;
        }

        if (hash_equals($record->otp, hash('sha256', $enteredOtp))) {
            // Mark as used
            Capsule::table('mod_sms_otp')
                ->where('id', $record->id)
                ->update(['verified_at' => date('Y-m-d H:i:s')]);
            return true;
        }

        // Increment attempts
        Capsule::table('mod_sms_otp')
            ->where('id', $record->id)
            ->increment('attempts');

        return false;
    }

    private function sendOTP(string $phone, string $otp): void {
        $api = new SMS\ApiClient(getModuleSettings());
        $api->sendSMS($phone, "Your verification code is: $otp. Valid for 5 minutes.");
    }

    public function resendOTP(string $phone): string {
        // Invalidate previous OTPs
        Capsule::table('mod_sms_otp')
            ->where('phone', $phone)
            ->whereNull('verified_at')
            ->update(['expires_at' => date('Y-m-d H:i:s')]);

        return $this->generateOTP($phone);
    }
}
```

### SMS in Hooks

```php
// Send SMS on specific events
add_hook('InvoiceCreation', 1, function($vars) {
    $client = Capsule::table('tblclients')->where('id', $vars['userid'])->first();
    if ($client->mobilephone) {
        $sms = new SMS\ApiClient(getModuleSettings());
        $sms->sendSMS($client->mobilephone,
            "Invoice #{$vars['invoiceid']} created. Amount: {$vars['amount']}");
    }
});

add_hook('TicketOpen', 1, function($vars) {
    $ticket = Capsule::table('tbltickets')->where('id', $vars['ticketid'])->first();
    $admins = Capsule::table('tbladmins')->where('supporter', 1)->get();

    foreach ($admins as $admin) {
        if ($admin->mobilephone) {
            $sms = new SMS\ApiClient(getModuleSettings());
            $sms->sendSMS($admin->mobilephone,
                "New Support Ticket: {$ticket->title}");
        }
    }
});
```

---

**Related Skills:**
- whmcs-notification-builder
- whmcs-hooks-development
- whmcs-security-hardening
