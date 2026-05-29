# WHMCS Mail Provider Development

## Overview

Mail providers allow WHMCS to send emails through custom email services like SendGrid, Mailgun, or AWS SES.

## Module Structure

```
modules/mail/YourMailProvider/
├── MailProvider.php
└── lang/
    └── english.php
```

## Provider Implementation

```php
<?php
// modules/mail/YourMailProvider/MailProvider.php

namespace WHMCS\Module\Mail\YourMailProvider;

use WHMCS\Mail\Contracts\ProviderInterface;
use WHMCS\Mail\Message;

class MailProvider implements ProviderInterface
{
    private string $apiKey;
    private string $fromEmail;
    private string $fromName;
    
    public function __construct()
    {
        $this->apiKey = \WHMCS\Config\Setting::getValue('YourMailProvider_api_key');
        $this->fromEmail = \WHMCS\Config\Setting::getValue('YourMailProvider_from_email');
        $this->fromName = \WHMCS\Config\Setting::getValue('YourMailProvider_from_name');
    }
    
    public static function getName(): string
    {
        return 'Your Mail Provider';
    }
    
    public static function getDescription(): string
    {
        return 'Send emails through Your Mail Provider API';
    }
    
    public static function getConfiguration(): array
    {
        return [
            'api_key' => [
                'Name' => 'API Key',
                'Type' => 'password',
                'Required' => true,
            ],
            'from_email' => [
                'Name' => 'Default From Email',
                'Type' => 'text',
                'Required' => true,
            ],
            'from_name' => [
                'Name' => 'Default From Name',
                'Type' => 'text',
                'Required' => true,
            ],
            'webhook_url' => [
                'Name' => 'Webhook URL (for bounces)',
                'Type' => 'text',
                'Required' => false,
            ],
        ];
    }
    
    public function send(Message $message): array
    {
        try {
            $payload = $this->buildPayload($message);
            $result = $this->apiCall('/messages/send', $payload);
            
            if ($result['success']) {
                return [
                    'success' => true,
                    'message_id' => $result['id'],
                ];
            }
            
            return [
                'success' => false,
                'error' => $result['error'] ?? 'Send failed',
            ];
            
        } catch (Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage(),
            ];
        }
    }
    
    public function validateConfiguration(array $config): array
    {
        $errors = [];
        
        if (empty($config['api_key'])) {
            $errors[] = 'API Key is required';
        }
        
        if (empty($config['from_email'])) {
            $errors[] = 'From Email is required';
        }
        
        if (!filter_var($config['from_email'], FILTER_VALIDATE_EMAIL)) {
            $errors[] = 'From Email is invalid';
        }
        
        return $errors;
    }
    
    private function buildPayload(Message $message): array
    {
        $payload = [
            'from' => [
                'email' => $message->getFromAddress() ?: $this->fromEmail,
                'name' => $message->getFromName() ?: $this->fromName,
            ],
            'to' => [],
            'subject' => $message->getSubject(),
            'text' => $message->getBody(),
        ];
        
        if ($message->getHtmlBody()) {
            $payload['html'] = $message->getHtmlBody();
        }
        
        // Add recipients
        foreach ($message->getRecipients() as $recipient) {
            $payload['to'][] = [
                'email' => $recipient['address'],
                'name' => $recipient['name'] ?? '',
            ];
        }
        
        // Add CC
        foreach ($message->getCc() as $cc) {
            $payload['cc'][] = [
                'email' => $cc['address'],
                'name' => $cc['name'] ?? '',
            ];
        }
        
        // Add BCC
        foreach ($message->getBcc() as $bcc) {
            $payload['bcc'][] = [
                'email' => $bcc['address'],
                'name' => $bcc['name'] ?? '',
            ];
        }
        
        // Add attachments
        foreach ($message->getAttachments() as $attachment) {
            $payload['attachments'][] = [
                'content' => base64_encode($attachment['data']),
                'filename' => $attachment['name'],
                'type' => $attachment['type'],
            ];
        }
        
        return $payload;
    }
    
    private function apiCall(string $endpoint, array $data): array
    {
        $url = 'https://api.yourprovider.com/v1' . $endpoint;
        
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
            CURLOPT_TIMEOUT => 30,
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        $result = json_decode($response, true);
        
        if ($httpCode >= 200 && $httpCode < 300) {
            return ['success' => true, 'id' => $result['id'] ?? ''];
        }
        
        return [
            'success' => false,
            'error' => $result['message'] ?? 'API error',
        ];
    }
}
```

## Message Interface

```php
<?php
namespace WHMCS\Mail;

class Message
{
    private string $subject;
    private string $body;
    private ?string $htmlBody = null;
    private array $recipients = [];
    private array $cc = [];
    private array $bcc = [];
    private array $attachments = [];
    private string $fromAddress;
    private string $fromName;
    private array $headers = [];
    
    public function setSubject(string $subject): self
    {
        $this->subject = $subject;
        return $this;
    }
    
    public function getSubject(): string
    {
        return $this->subject;
    }
    
    public function setBody(string $body): self
    {
        $this->body = $body;
        return $this;
    }
    
    public function getBody(): string
    {
        return $this->body;
    }
    
    public function setHtmlBody(string $html): self
    {
        $this->htmlBody = $html;
        return $this;
    }
    
    public function getHtmlBody(): ?string
    {
        return $this->htmlBody;
    }
    
    public function addRecipient(string $address, string $name = ''): self
    {
        $this->recipients[] = ['address' => $address, 'name' => $name];
        return $this;
    }
    
    public function getRecipients(): array
    {
        return $this->recipients;
    }
    
    public function addCc(string $address, string $name = ''): self
    {
        $this->cc[] = ['address' => $address, 'name' => $name];
        return $this;
    }
    
    public function getCc(): array
    {
        return $this->cc;
    }
    
    public function addBcc(string $address, string $name = ''): self
    {
        $this->bcc[] = ['address' => $address, 'name' => $name];
        return $this;
    }
    
    public function getBcc(): array
    {
        return $this->bcc;
    }
    
    public function addAttachment(string $name, string $data, string $type): self
    {
        $this->attachments[] = [
            'name' => $name,
            'data' => $data,
            'type' => $type,
        ];
        return $this;
    }
    
    public function getAttachments(): array
    {
        return $this->attachments;
    }
    
    public function setFrom(string $address, string $name = ''): self
    {
        $this->fromAddress = $address;
        $this->fromName = $name;
        return $this;
    }
    
    public function getFromAddress(): string
    {
        return $this->fromAddress;
    }
    
    public function getFromName(): string
    {
        return $this->fromName;
    }
    
    public function addHeader(string $name, string $value): self
    {
        $this->headers[$name] = $value;
        return $this;
    }
    
    public function getHeaders(): array
    {
        return $this->headers;
    }
}
```

## Best Practices

1. **Validate configuration** - Check all settings on save
2. **Handle API errors** - Return proper error messages
3. **Support attachments** - Include attachment handling
4. **Implement webhook support** - Handle bounces and tracking
5. **Log all sends** - Track email delivery

## Related Documentation

- [WHMCS Email Functions](/docs/whmcs-functions-email.md)