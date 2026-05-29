# WHMCS SMS Automation Workflow

## Overview
This workflow implements SMS automation for WHMCS including transactional SMS, notifications, and 2FA.

## Prerequisites
- WHMCS installation
- SMS provider account (Twilio, Nexmo/Vonage, etc.)
- SMS API credentials

## Step-by-Step Process

### Step 1: Create SMS Provider Interface
```php
<?php
// /includes/sms/SMSProviderInterface.php

namespace WHMCS\SMS;

interface SMSProviderInterface
{
    public function send(string $to, string $message, array $options = []): array;
    public function sendBatch(array $recipients, string $message): array;
    public function getBalance(): float;
    public function getMessageStatus(string $messageId): string;
}
```

### Step 2: Create SMS Manager
```php
<?php
// /includes/sms/SMSManager.php

namespace WHMCS\SMS;

class SMSManager
{
    private $provider;
    private $enabled = false;

    public function __construct()
    {
        $this->loadProvider();
    }

    private function loadProvider()
    {
        $providerType = getConfig('sms_provider');

        switch ($providerType) {
            case 'twilio':
                $this->provider = new TwilioProvider();
                $this->enabled = true;
                break;
            case 'nexmo':
                $this->provider = new NexmoProvider();
                $this->enabled = true;
                break;
            default:
                $this->enabled = false;
        }
    }

    /**
     * Send SMS
     */
    public function send(int $clientId, string $template, array $data = [], array $options = []): array
    {
        if (!$this->enabled) {
            return ['success' => false, 'error' => 'SMS disabled'];
        }

        // Get client's phone number
        $client = Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();

        if (!$client || empty($client->phonenumber)) {
            return ['success' => false, 'error' => 'No phone number'];
        }

        // Render message from template
        $message = $this->renderTemplate($template, $data, $client);

        // Send via provider
        $result = $this->provider->send($client->phonenumber, $message, $options);

        // Log the SMS
        $this->logSMS($clientId, $template, $message, $result);

        return $result;
    }

    /**
     * Send to multiple clients
     */
    public function sendBulk(array $clientIds, string $template, array $data = []): array
    {
        $results = [
            'sent' => 0,
            'failed' => 0,
            'errors' => []
        ];

        foreach ($clientIds as $clientId) {
            $result = $this->send($clientId, $template, $data);

            if ($result['success']) {
                $results['sent']++;
            } else {
                $results['failed']++;
                $results['errors'][] = ['client_id' => $clientId, 'error' => $result['error']];
            }

            // Rate limiting
            usleep(100000);
        }

        return $results;
    }

    private function renderTemplate(string $template, array $data, $client): string
    {
        $templates = [
            'payment_received' => "Hi {$client->firstname}, payment of {$data['amount']} received. Thank you!",
            'service_suspended' => "Hi {$client->firstname}, your service has been suspended. Please log in to resolve.",
            'service_activated' => "Hi {$client->firstname}, your service is now active! Welcome aboard.",
            'renewal_reminder' => "Hi {$client->firstname}, your service renews on {$data['due_date']}. Total: {$data['amount']}",
            'support_reply' => "Hi {$client->firstname}, new reply to your ticket #{$data['ticket_id']}: {$data['message']}",
            'verification_code' => "Your verification code is: {$data['code']}. Valid for 10 minutes.",
            'security_alert' => "Security alert: {$data['message']}. If not you, contact support immediately."
        ];

        return $templates[$template] ?? $template;
    }

    private function logSMS(int $clientId, string $template, string $message, array $result)
    {
        Capsule::table('mod_sms_log')->insert([
            'client_id' => $clientId,
            'template' => $template,
            'message' => substr($message, 0, 500),
            'success' => $result['success'] ? 1 : 0,
            'message_id' => $result['message_id'] ?? null,
            'error' => $result['error'] ?? null,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

### Step 3: Create SMS Provider Implementations
```php
<?php
// /includes/sms/providers/TwilioProvider.php

class TwilioProvider implements SMSProviderInterface
{
    private $sid;
    private $token;
    private $from;

    public function __construct()
    {
        $this->sid = getConfig('twilio_sid');
        $this->token = getConfig('twilio_token');
        $this->from = getConfig('twilio_from');
    }

    public function send(string $to, string $message, array $options = []): array
    {
        $url = "https://api.twilio.com/2010-04-01/Accounts/{$this->sid}/Messages.json";

        $data = [
            'To' => $this->formatPhoneNumber($to),
            'From' => $this->from,
            'Body' => $message
        ];

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($data),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_USERPWD => "{$this->sid}:{$this->token}"
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $result = json_decode($response, true);

        if ($httpCode >= 200 && $httpCode < 300) {
            return [
                'success' => true,
                'message_id' => $result['sid']
            ];
        }

        return [
            'success' => false,
            'error' => $result['message'] ?? 'Unknown error'
        ];
    }

    public function sendBatch(array $recipients, string $message): array
    {
        $results = [];

        foreach ($recipients as $to) {
            $results[] = $this->send($to, $message);
            usleep(100000); // Rate limiting
        }

        return $results;
    }

    public function getBalance(): float
    {
        $url = "https://api.twilio.com/2010-04-01/Accounts/{$this->sid}/Balance.json";

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_USERPWD => "{$this->sid}:{$this->token}"
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        $result = json_decode($response, true);

        return $result['balance'] ?? 0;
    }

    public function getMessageStatus(string $messageId): string
    {
        $url = "https://api.twilio.com/2010-04-01/Accounts/{$this->sid}/Messages/{$messageId}.json";

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_USERPWD => "{$this->sid}:{$this->token}"
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        $result = json_decode($response, true);

        return $result['status'] ?? 'unknown';
    }

    private function formatPhoneNumber(string $phone): string
    {
        $phone = preg_replace('/[^0-9+]/', '', $phone);

        if (strpos($phone, '+') !== 0 && strlen($phone) >= 10) {
            $phone = '+1' . ltrim($phone, '1');
        }

        return $phone;
    }
}
```

### Step 4: Create SMS Hooks
```php
<?php
// /includes/hooks/sms_hooks.php

use WHMCS\SMS\SMSManager;

$smsManager = new SMSManager();

// Payment received
add_hook('InvoicePaid', 1, function($vars) use ($smsManager) {
    $invoice = getInvoice($vars['invoiceid']);

    if (clientHasSMSEnabled($invoice['userid'])) {
        $smsManager->send($invoice['userid'], 'payment_received', [
            'amount' => formatCurrency($invoice['total'])
        ], ['priority' => 'normal']);
    }
});

// Service suspended
add_hook('ServiceSuspended', 1, function($vars) use ($smsManager) {
    if (clientHasSMSEnabled($vars['userid'])) {
        $smsManager->send($vars['userid'], 'service_suspended', [
            'service_id' => $vars['serviceid']
        ], ['priority' => 'high']);
    }
});

// Service activated
add_hook('AfterServiceCreate', 1, function($vars) use ($smsManager) {
    $service = Capsule::table('tblhosting')
        ->where('id', $vars['serviceid'])
        ->first();

    if ($service->domainstatus === 'Active' && clientHasSMSEnabled($vars['userid'])) {
        $smsManager->send($vars['userid'], 'service_activated');
    }
});

// Support ticket reply
add_hook('TicketReply', 1, function($vars) use ($smsManager) {
    $ticket = Capsule::table('tbltickets')
        ->where('id', $vars['ticketid'])
        ->first();

    if (clientHasSMSEnabled($ticket->userid)) {
        $smsManager->send($ticket->userid, 'support_reply', [
            'ticket_id' => $vars['ticketid'],
            'message' => substr($vars['message'], 0, 50) . '...'
        ]);
    }
});
```

### Step 5: Create 2FA SMS Implementation
```php
<?php
// /includes/sms/SMS2FA.php

class SMS2FA
{
    private $smsManager;

    public function __construct()
    {
        $this->smsManager = new SMSManager();
    }

    /**
     * Generate and send verification code
     */
    public function sendVerificationCode(int $clientId): array
    {
        // Generate 6-digit code
        $code = str_pad((string)random_int(0, 999999), 6, '0', STR_PAD_LEFT);

        // Store code in database
        Capsule::table('mod_sms_2fa')->insert([
            'client_id' => $clientId,
            'code' => password_hash($code, PASSWORD_DEFAULT),
            'expires_at' => date('Y-m-d H:i:s', strtotime('+10 minutes')),
            'attempts' => 0,
            'created_at' => date('Y-m-d H:i:s')
        ]);

        // Send SMS
        $result = $this->smsManager->send($clientId, 'verification_code', [
            'code' => $code
        ]);

        return [
            'success' => $result['success'],
            'expires_in' => 600 // 10 minutes
        ];
    }

    /**
     * Verify code
     */
    public function verifyCode(int $clientId, string $code): array
    {
        $record = Capsule::table('mod_sms_2fa')
            ->where('client_id', $clientId)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->orderBy('created_at', 'DESC')
            ->first();

        if (!$record) {
            return ['success' => false, 'error' => 'Code expired or not found'];
        }

        // Check attempts
        if ($record->attempts >= 3) {
            Capsule::table('mod_sms_2fa')
                ->where('id', $record->id)
                ->update(['status' => 'locked']);

            return ['success' => false, 'error' => 'Too many attempts'];
        }

        // Verify code
        if (password_verify($code, $record->code)) {
            // Mark as verified
            Capsule::table('mod_sms_2fa')
                ->where('id', $record->id)
                ->update(['verified' => 1, 'verified_at' => date('Y-m-d H:i:s')]);

            return ['success' => true];
        }

        // Increment attempts
        Capsule::table('mod_sms_2fa')
            ->where('id', $record->id)
            ->increment('attempts');

        return ['success' => false, 'error' => 'Invalid code'];
    }
}
```

### Step 6: SMS Marketing Automation
```php
<?php
// SMS marketing campaigns

class SMSMarketing
{
    private $smsManager;

    public function __construct()
    {
        $this->smsManager = new SMSManager();
    }

    /**
     * Send campaign to segment
     */
    public function sendCampaign(string $segment, string $message, array $options = []): array
    {
        $clients = $this->getSegmentClients($segment);

        $results = [
            'total' => count($clients),
            'sent' => 0,
            'failed' => 0,
            'opt_outs' => 0
        ];

        foreach ($clients as $client) {
            // Check opt-out
            if ($this->isOptedOut($client->id)) {
                $results['opt_outs']++;
                continue;
            }

            $result = $this->smsManager->send($client->id, 'campaign', [
                'message' => $message
            ]);

            if ($result['success']) {
                $results['sent']++;
            } else {
                $results['failed']++;
            }

            // Rate limiting for SMS
            sleep(1);
        }

        return $results;
    }

    private function getSegmentClients(string $segment): array
    {
        $query = Capsule::table('tblclients')
            ->whereNotNull('phonenumber')
            ->where('phonenumber', '!=', '');

        switch ($segment) {
            case 'active_services':
                $query->whereExists(function($q) {
                    $q->select(Capsule::raw(1))
                        ->from('tblhosting')
                        ->whereRaw('tblhosting.userid = tblclients.id')
                        ->where('domainstatus', 'Active');
                });
                break;
            case 'recent_customers':
                $query->where('datecreated', '>=', date('Y-m-d', strtotime('-30 days')));
                break;
            case 'vip':
                $query->where('groupid', 1); // VIP group
                break;
        }

        return $query->get();
    }

    private function isOptedOut(int $clientId): bool
    {
        return Capsule::table('mod_sms_optout')
            ->where('client_id', $clientId)
            ->exists();
    }
}
```

## SMS Automation Best Practices

1. **Get consent** - Only send to opted-in clients
2. **Keep messages short** - SMS is limited to 160 characters
3. **Time messages well** - Avoid early morning/late night
4. **Provide opt-out** - Easy way to unsubscribe
5. **Monitor costs** - Track SMS usage and budget
6. **Use templates** - Consistent messaging

## Related Workflows
- [WHMCS Email Automation](./whmcs-email-automation.md)
- [WHMCS Notification Automation](./whmcs-notification-automation.md)
- [WHMCS Two-Factor Authentication](./whmcs-two-factor-auth.md)