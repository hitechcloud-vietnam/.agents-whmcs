# WHMCS Email Provider Setup Workflow

## Overview
This workflow covers integrating third-party email providers with WHMCS.

## Step 1: Email Provider Integration

```php
<?php
// src/Service/EmailProviderService.php

namespace WHMCS\Module\Addon\YourModule\Service;

class EmailProviderService
{
    private $apiKey;
    private $fromEmail;
    private $fromName;
    private $provider;

    public function __construct(string $provider, string $apiKey, string $fromEmail, string $fromName)
    {
        $this->provider = $provider;
        $this->apiKey = $apiKey;
        $this->fromEmail = $fromEmail;
        $this->fromName = $fromName;
    }

    public function send(string $to, string $subject, string $html, string $text = ''): array
    {
        return match ($this->provider) {
            'sendgrid' => $this->sendViaSendGrid($to, $subject, $html, $text),
            'mailgun' => $this->sendViaMailgun($to, $subject, $html, $text),
            'ses' => $this->sendViaSES($to, $subject, $html, $text),
            default => throw new \Exception("Unknown provider: {$this->provider}")
        };
    }

    private function sendViaSendGrid(string $to, string $subject, string $html, string $text): array
    {
        $ch = curl_init('https://api.sendgrid.com/v3/mail/send');
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json'
            ],
            CURLOPT_POSTFIELDS => json_encode([
                'personalizations' => [['to' => [['email' => $to]]]],
                'from' => ['email' => $this->fromEmail, 'name' => $this->fromName],
                'subject' => $subject,
                'content' => [
                    ['type' => 'text/html', 'value' => $html],
                    ['type' => 'text/plain', 'value' => $text]
                ]
            ]),
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return [
            'success' => $httpCode >= 200 && $httpCode < 300,
            'http_code' => $httpCode
        ];
    }

    private function sendViaMailgun(string $to, string $subject, string $html, string $text): array
    {
        $domain = Capsule::config('mailgun_domain');

        $ch = curl_init("https://api.mailgun.net/v3/$domain/messages");
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_USERPWD => 'api:' . $this->apiKey,
            CURLOPT_POSTFIELDS => [
                'from' => "$this->fromName <$this->fromEmail>",
                'to' => $to,
                'subject' => $subject,
                'html' => $html,
                'text' => $text
            ],
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = json_decode(curl_exec($ch), true);
        curl_close($ch);

        return [
            'success' => isset($response['id']),
            'message_id' => $response['id'] ?? null
        ];
    }

    private function sendViaSES(string $to, string $subject, string $html, string $text): array
    {
        // AWS SES implementation
        // Requires AWS SDK or direct API calls
        return ['success' => true];
    }

    public function getStats(): array
    {
        return match ($this->provider) {
            'sendgrid' => $this->getSendGridStats(),
            'mailgun' => $this->getMailgunStats(),
            default => []
        };
    }

    private function getSendGridStats(): array
    {
        $ch = curl_init('https://api.sendgrid.com/v3/stats');
        curl_setopt_array($ch, [
            CURLOPT_HTTPHEADER => ['Authorization: Bearer ' . $this->apiKey],
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = json_decode(curl_exec($ch), true);
        curl_close($ch);

        return $response;
    }

    private function getMailgunStats(): array
    {
        $domain = Capsule::config('mailgun_domain');

        $ch = curl_init("https://api.mailgun.net/v3/$domain/stats");
        curl_setopt_array($ch, [
            CURLOPT_USERPWD => 'api:' . $this->apiKey,
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = json_decode(curl_exec($ch), true);
        curl_close($ch);

        return $response;
    }
}
```

## Verification Checklist

- [ ] Email provider service implemented
- [ ] SendGrid integration working
- [ ] Mailgun integration working
- [ ] Stats retrieval working
- [ ] Test email sent successfully
