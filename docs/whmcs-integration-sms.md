# WHMCS SMS Integration

Complete guide for integrating WHMCS with SMS providers.

## Overview

Add SMS notifications for order confirmations, payment reminders, and support updates.

## Twilio Integration

```php
<?php
/**
 * Twilio SMS client
 */
class TwilioSMSClient
{
    private string $accountSid;
    private string $authToken;
    private string $fromNumber;
    
    public function __construct(array $config)
    {
        $this->accountSid = $config['account_sid'];
        $this->authToken = $config['auth_token'];
        $this->fromNumber = $config['from_number'];
    }
    
    /**
     * Send SMS
     */
    public function send(string $to, string $message): array
    {
        $data = [
            'To' => $this->formatPhoneNumber($to),
            'From' => $this->fromNumber,
            'Body' => $message,
        ];
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => "https://api.twilio.com/2010-04-01/Accounts/{$this->accountSid}/Messages.json",
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($data),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_USERPWD => "{$this->accountSid}:{$this->authToken}",
        ]);
        
        $response = json_decode(curl_exec($ch), true);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return [
            'success' => $httpCode === 201,
            'message_id' => $response['sid'] ?? null,
            'status' => $response['status'] ?? null,
            'error' => $response['message'] ?? null,
        ];
    }
    
    /**
     * Format phone number to E.164
     */
    private function formatPhoneNumber(string $phone): string
    {
        $phone = preg_replace('/[^0-9+]/', '', $phone);
        
        if (strpos($phone, '+') !== 0) {
            if (strpos($phone, '1') === 0 && strlen($phone) === 11) {
                return '+' . $phone;
            }
            return '+1' . $phone;
        }
        
        return $phone;
    }
    
    /**
     * Send templated SMS
     */
    public function sendTemplate(string $to, string $templateId, array $variables): array
    {
        $template = $this->getTemplate($templateId);
        $message = $this->interpolate($template, $variables);
        
        return $this->send($to, $message);
    }
    
    /**
     * Get message template
     */
    private function getTemplate(string $templateId): string
    {
        return Capsule::table('mod_sms_templates')
            ->where('id', $templateId)
            ->first()->content ?? '';
    }
    
    /**
     * Replace variables in template
     */
    private function interpolate(string $template, array $variables): string
    {
        foreach ($variables as $key => $value) {
            $template = str_replace('{{' . $key . '}}', $value, $template);
        }
        return $template;
    }
}
```

### Twilio Webhook Handler

```php
<?php
/**
 * Handle Twilio status callbacks
 */
add_hook('TwilioStatusCallback', 1, function($vars) {
    $messageSid = $vars['MessageSid'];
    $status = $vars['MessageStatus'];
    
    Capsule::table('mod_sms_log')
        ->where('message_sid', $messageSid)
        ->update([
            'status' => $status,
            'status_updated_at' => date('Y-m-d H:i:s'),
            'error_code' => $vars['ErrorCode'] ?? null,
        ]);
    
    // Handle failed messages
    if ($status === 'failed' || $status === 'undelivered') {
        logActivity("SMS failed: {$messageSid} - {$vars['ErrorCode']}");
        
        // Optionally retry or notify admin
        $message = Capsule::table('mod_sms_log')
            ->where('message_sid', $messageSid)
            ->first();
        
        if ($message && $message->retry_count < 3) {
            // Queue for retry
            Capsule::table('mod_sms_queue')->insert([
                'recipient' => $message->recipient,
                'message' => $message->message,
                'retry_count' => $message->retry_count + 1,
                'scheduled_at' => date('Y-m-d H:i:s', strtotime('+5 minutes')),
            ]);
        }
    }
});
```

## AWS SNS Integration

```php
<?php
/**
 * AWS SNS SMS client
 */
class AWSSNSClient
{
    private string $accessKey;
    private string $secretKey;
    private string $region;
    private string $senderId;
    
    public function __construct(array $config)
    {
        $this->accessKey = $config['access_key'];
        $this->secretKey = $config['secret_key'];
        $this->region = $config['region'] ?? 'us-east-1';
        $this->senderId = $config['sender_id'] ?? '';
    }
    
    /**
     * Send SMS
     */
    public function send(string $phone, string $message): array
    {
        $endpoint = "https://sns.{$this->region}.amazonaws.com/";
        
        $payload = [
            'Action' => 'Publish',
            'PhoneNumber' => $this->formatPhoneNumber($phone),
            'Message' => $message,
            'MessageAttributes.entry.1.Name' => 'AWS.SNS.SMS.SenderID',
            'MessageAttributes.entry.1.Value.DataType' => 'String',
            'MessageAttributes.entry.1.Value.StringValue' => $this->senderId,
            'MessageAttributes.entry.2.Name' => 'AWS.SNS.SMS.SMSType',
            'MessageAttributes.entry.2.Value.DataType' => 'String',
            'MessageAttributes.entry.2.Value.StringValue' => 'Transactional',
            'Version' => '2010-03-31',
        ];
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $endpoint,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($payload),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: AWS4-HMAC-SHA256',
                'X-Amz-Date: ' . gmdate('Ymd\THis\Z'),
            ],
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return [
            'success' => strpos($response, '<PublishResult>') !== false,
            'message_id' => $this->extractMessageId($response),
        ];
    }
    
    private function formatPhoneNumber(string $phone): string
    {
        $phone = preg_replace('/[^0-9+]/', '', $phone);
        return strpos($phone, '+') === 0 ? $phone : '+' . $phone;
    }
    
    private function extractMessageId(string $response): ?string
    {
        preg_match('/<MessageId>(.*?)<\/MessageId>/', $response, $matches);
        return $matches[1] ?? null;
    }
}
```

## WHMCS SMS Hook

```php
<?php
/**
 * SMS notification on order
 */
add_hook('OrderPaid', 1, function($vars) {
    $orderId = $vars['order_id'];
    
    $order = Capsule::table('tblorders')
        ->where('id', $orderId)
        ->first();
    
    $client = Capsule::table('tblclients')
        ->where('id', $order->userid)
        ->first();
    
    if ($client && !empty($client->phonenumber)) {
        $sms = new TwilioSMSClient([
            'account_sid' => TWILIO_SID,
            'auth_token' => TWILIO_TOKEN,
            'from_number' => TWILIO_FROM,
        ]);
        
        $sms->sendTemplate($client->phonenumber, 'order_confirmation', [
            'order_id' => $orderId,
            'domain' => $order->domain,
            'amount' => $order->total,
        ]);
    }
});

/**
 * SMS reminder for overdue invoices
 */
function sendOverdueInvoiceSMS(): void
{
    $overdueInvoices = Capsule::table('tblinvoices')
        ->where('status', 'Overdue')
        ->where('duedate', '<', date('Y-m-d', strtotime('-3 days')))
        ->get();
    
    foreach ($overdueInvoices as $invoice) {
        $client = Capsule::table('tblclients')
            ->where('id', $invoice->userid)
            ->first();
        
        if ($client && !empty($client->phonenumber)) {
            $sms = new TwilioSMSClient([
                'account_sid' => TWILIO_SID,
                'auth_token' => TWILIO_TOKEN,
                'from_number' => TWILIO_FROM,
            ]);
            
            $message = "Reminder: Invoice #{$invoice->id} for \${$invoice->total} is overdue. Please pay at: " . 
                       WHMCS_URL . "/viewinvoice.php?id={$invoice->id}";
            
            $sms->send($client->phonenumber, $message);
            
            // Log the SMS
            Capsule::table('mod_sms_log')->insert([
                'invoice_id' => $invoice->id,
                'recipient' => $client->phonenumber,
                'message' => substr($message, 0, 160),
                'status' => 'queued',
                'created_at' => date('Y-m-d H:i:s'),
            ]);
        }
    }
}
```

## SMS Queue Management

```php
<?php
/**
 * Process SMS queue
 */
function processSMSQueue(): int
{
    $pending = Capsule::table('mod_sms_queue')
        ->where('status', 'pending')
        ->where('scheduled_at', '<=', date('Y-m-d H:i:s'))
        ->where('retry_count', '<', 3)
        ->limit(50)
        ->get();
    
    $sms = new TwilioSMSClient([
        'account_sid' => TWILIO_SID,
        'auth_token' => TWILIO_TOKEN,
        'from_number' => TWILIO_FROM,
    ]);
    
    $processed = 0;
    
    foreach ($pending as $item) {
        $result = $sms->send($item->recipient, $item->message);
        
        if ($result['success']) {
            Capsule::table('mod_sms_queue')
                ->where('id', $item->id)
                ->update([
                    'status' => 'sent',
                    'sent_at' => date('Y-m-d H:i:s'),
                    'message_id' => $result['message_id'],
                ]);
        } else {
            Capsule::table('mod_sms_queue')
                ->where('id', $item->id)
                ->update([
                    'retry_count' => $item->retry_count + 1,
                    'last_error' => $result['error'] ?? 'Unknown error',
                    'scheduled_at' => date('Y-m-d H:i:s', strtotime('+10 minutes')),
                ]);
        }
        
        $processed++;
    }
    
    return $processed;
}
```

## SMS Templates

```php
<?php
/**
 * Create SMS template
 */
function createSMSTemplate(string $name, string $content, string $description = ''): int
{
    Capsule::table('mod_sms_templates')->insert([
        'name' => $name,
        'content' => $content,
        'description' => $description,
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    return Capsule::getPdo()->lastInsertId();
}

/**
 * Available templates
 */
$templates = [
    'order_confirmation' => 'Your order #{{order_id}} for {{domain}} (${{amount}}) has been confirmed.',
    'payment_received' => 'Payment received for invoice #{{invoice_id}}. Thank you!',
    'service_suspended' => 'Your service {{domain}} has been suspended. Please resolve payment.',
    'support_reply' => 'New reply to ticket #{{ticket_id}}: {{reply_preview}}',
    'two_factor_code' => 'Your verification code is: {{code}}. Valid for 10 minutes.',
];
```

## Best Practices

1. **Format phone numbers** - Use E.164 format
2. **Handle delivery status** - Track message status
3. **Implement retries** - Retry failed messages
4. **Rate limiting** - Respect provider limits
5. **Opt-out handling** - Respect unsubscribe requests
6. **Log all messages** - Keep audit trail

## Related Documentation

- [whmcs-integration-email.md](whmcs-integration-email.md)
- [whmcs-integration-webhooks.md](whmcs-integration-webhooks.md)
