# WHMCS SendGrid Integration Workflow

## Overview
This workflow implements SendGrid email integration for WHMCS.

## Prerequisites
- WHMCS with SendGrid SDK
- SendGrid API key
- Admin access

## Step-by-Step Process

### Step 1: SendGrid Integration
```php
<?php
// /includes/email/SendGridIntegration.php

class SendGridIntegration {
    private $apiKey;

    public function __construct()
    {
        $this->apiKey = getConfig('sendgrid_api_key');
    }

    /**
     * Send email
     */
    public function sendEmail(string $to, string $subject, string $html, array $attachments = []): array
    {
        $data = [
            'personalizations' => [[
                'to' => [['email' => $to]]
            ]],
            'from' => ['email' => getConfig('from_email')],
            'subject' => $subject,
            'content' => [[
                'type' => 'text/html',
                'value' => $html
            ]]
        ];

        if (!empty($attachments)) {
            $data['attachments'] = $attachments;
        }

        $ch = curl_init('https://api.sendgrid.com/v3/mail/send');

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json'
            ],
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return [
            'success' => $httpCode === 202,
            'http_code' => $httpCode
        ];
    }
}
```

## Related Workflows
- [WHMCS Mailgun Integration](./whmcs-mailgun-integration.md)
- [WHMCS Email Automation](./whmcs-email-automation.md)