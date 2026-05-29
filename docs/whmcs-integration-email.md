# WHMCS Email Integration

Complete guide for integrating WHMCS with email services.

## Overview

Connect WHMCS to email providers for transactional emails, marketing campaigns, and support notifications.

## SMTP Configuration

### Custom SMTP Client

```php
<?php
/**
 * Custom SMTP email sender
 */
class SMTPEmailClient
{
    private string $host;
    private int $port;
    private string $username;
    private string $password;
    private bool $useTLS;
    
    public function __construct(array $config)
    {
        $this->host = $config['host'];
        $this->port = $config['port'] ?? 587;
        $this->username = $config['username'];
        $this->password = $config['password'];
        $this->useTLS = $config['use_tls'] ?? true;
    }
    
    /**
     * Send email
     */
    public function send(array $email): bool
    {
        $socket = fsockopen(
            $this->host,
            $this->port,
            $errno,
            $errstr,
            30
        );
        
        if (!$socket) {
            throw new Exception("Connection failed: {$errstr}");
        }
        
        // Read greeting
        $response = fgets($socket, 515);
        
        // Send EHLO
        $this->sendCommand($socket, "EHLO " . gethostname());
        $this->readResponse($socket);
        
        // Start TLS if needed
        if ($this->useTLS) {
            $this->sendCommand($socket, "STARTTLS");
            $this->readResponse($socket);
            
            stream_socket_enable_crypto($socket, true, STREAM_CRYPTO_METHOD_TLS_CLIENT);
            
            $this->sendCommand($socket, "EHLO " . gethostname());
            $this->readResponse($socket);
        }
        
        // Authenticate
        $this->sendCommand($socket, "AUTH LOGIN");
        $this->readResponse($socket);
        
        $this->sendCommand($socket, base64_encode($this->username));
        $this->readResponse($socket);
        
        $this->sendCommand($socket, base64_encode($this->password));
        $this->readResponse($socket);
        
        // Send email
        $this->sendCommand($socket, "MAIL FROM:<{$email['from_email']}>");
        $this->readResponse($socket);
        
        foreach ($email['to'] as $recipient) {
            $this->sendCommand($socket, "RCPT TO:<{$recipient}>");
            $this->readResponse($socket);
        }
        
        $this->sendCommand($socket, "DATA");
        $this->readResponse($socket);
        
        // Build message
        $message = $this->buildMessage($email);
        $this->sendCommand($socket, $message . "\r\n.");
        $this->readResponse($socket);
        
        // Quit
        $this->sendCommand($socket, "QUIT");
        $this->readResponse($socket);
        
        fclose($socket);
        
        return true;
    }
    
    /**
     * Build email message
     */
    private function buildMessage(array $email): string
    {
        $headers = [
            'From' => "{$email['from_name']} <{$email['from_email']}>",
            'To' => implode(', ', $email['to']),
            'Subject' => $email['subject'],
            'MIME-Version' => '1.0',
            'Content-Type' => 'text/html; charset=UTF-8',
            'Date' => date('r'),
        ];
        
        if (!empty($email['reply_to'])) {
            $headers['Reply-To'] = $email['reply_to'];
        }
        
        $headerString = '';
        foreach ($headers as $key => $value) {
            $headerString .= "{$key}: {$value}\r\n";
        }
        
        return $headerString . "\r\n" . $email['body'];
    }
    
    private function sendCommand($socket, string $command): void
    {
        fwrite($socket, $command . "\r\n");
    }
    
    private function readResponse($socket): string
    {
        $response = '';
        while ($line = fgets($socket, 515)) {
            $response .= $line;
            if (substr($line, 3, 1) === ' ') break;
        }
        return $response;
    }
}
```

## SendGrid Integration

```php
<?php
/**
 * SendGrid email integration
 */
class SendGridEmailClient
{
    private string $apiKey;
    
    public function __construct(string $apiKey)
    {
        $this->apiKey = $apiKey;
    }
    
    /**
     * Send email via SendGrid API
     */
    public function send(array $email): array
    {
        $data = [
            'personalizations' => [[
                'to' => array_map(fn($e) => ['email' => $e], $email['to']),
                'subject' => $email['subject'],
            ]],
            'from' => [
                'email' => $email['from_email'],
                'name' => $email['from_name'] ?? '',
            ],
            'content' => [[
                'type' => 'text/html',
                'value' => $email['body'],
            ]],
        ];
        
        if (!empty($email['reply_to'])) {
            $data['reply_to'] = ['email' => $email['reply_to']];
        }
        
        if (!empty($email['cc'])) {
            $data['personalizations'][0]['cc'] = array_map(
                fn($e) => ['email' => $e],
                $email['cc']
            );
        }
        
        if (!empty($email['attachments'])) {
            $data['attachments'] = $email['attachments'];
        }
        
        $ch = curl_init('https://api.sendgrid.com/v3/mail/send');
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return [
            'success' => $httpCode >= 200 && $httpCode < 300,
            'status_code' => $httpCode,
        ];
    }
    
    /**
     * Send templated email
     */
    public function sendTemplate(string $templateId, array $email, array $substitutions): array
    {
        $data = [
            'personalizations' => [[
                'to' => array_map(fn($e) => ['email' => $e], $email['to']),
                'subject' => $email['subject'],
                'dynamic_template_data' => $substitutions,
            ]],
            'from' => [
                'email' => $email['from_email'],
                'name' => $email['from_name'] ?? '',
            ],
            'template_id' => $templateId,
        ];
        
        $ch = curl_init('https://api.sendgrid.com/v3/mail/send');
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return [
            'success' => $httpCode >= 200 && $httpCode < 300,
            'status_code' => $httpCode,
        ];
    }
}
```

## Mailgun Integration

```php
<?php
/**
 * Mailgun email integration
 */
class MailgunEmailClient
{
    private string $apiKey;
    private string $domain;
    
    public function __construct(string $apiKey, string $domain)
    {
        $this->apiKey = $apiKey;
        $this->domain = $domain;
    }
    
    /**
     * Send email via Mailgun API
     */
    public function send(array $email): array
    {
        $data = [
            'from' => "{$email['from_name']} <{$email['from_email']}@{$this->domain}>",
            'to' => implode(',', $email['to']),
            'subject' => $email['subject'],
            'html' => $email['body'],
        ];
        
        if (!empty($email['cc'])) {
            $data['cc'] = implode(',', $email['cc']);
        }
        
        if (!empty($email['reply_to'])) {
            $data['h:Reply-To'] = $email['reply_to'];
        }
        
        $ch = curl_init("https://api.mailgun.net/v3/{$this->domain}/messages");
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $data,
            CURLOPT_USERPWD => 'api:' . $this->apiKey,
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        $response = json_decode(curl_exec($ch), true);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return [
            'success' => $httpCode === 200,
            'response' => $response,
        ];
    }
    
    /**
     * Send batch email
     */
    public function sendBatch(array $recipients, array $email): array
    {
        $data = [
            'from' => "{$email['from_name']} <{$email['from_email']}@{$this->domain}>",
            'subject' => $email['subject'],
            'html' => $email['body'],
            'recipient-variables' => json_encode(
                array_reduce($recipients, function($carry, $recipient) {
                    $carry[$recipient] = ['email' => $recipient];
                    return $carry;
                }, [])
            ),
        ];
        
        // Add recipients as batch
        foreach ($recipients as $recipient) {
            $data['to'][] = $recipient;
        }
        
        $ch = curl_init("https://api.mailgun.net/v3/{$this->domain}/messages");
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $data,
            CURLOPT_USERPWD => 'api:' . $this->apiKey,
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        $response = json_decode(curl_exec($ch), true);
        curl_close($ch);
        
        return $response;
    }
    
    /**
     * Get email events
     */
    public function getEvents(string $messageId): array
    {
        $ch = curl_init("https://api.mailgun.net/v3/{$this->domain}/events");
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_USERPWD => 'api:' . $this->apiKey,
        ]);
        
        $response = json_decode(curl_exec($ch), true);
        curl_close($ch);
        
        return $response['items'] ?? [];
    }
}
```

## Email Template Hook

```php
<?php
/**
 * Custom email hook for sending
 */
add_hook('EmailPreSend', 1, function($vars) {
    $email = $vars['email'];
    
    // Use SendGrid for marketing emails
    if (strpos($email['subject'], '[Marketing]') !== false) {
        $sendgrid = new SendGridEmailClient(SENDGRID_API_KEY);
        
        $result = $sendgrid->send([
            'to' => [$email['mailto']],
            'from_email' => $email['mailfrom'],
            'from_name' => $email['fromname'],
            'subject' => $email['subject'],
            'body' => $email['body'],
        ]);
        
        // Return to skip default sending
        if ($result['success']) {
            return ['skip' => true, 'message_id' => $result['message_id']];
        }
    }
    
    return null; // Continue with default sending
});

/**
 * Log email after sending
 */
add_hook('EmailSent', 1, function($vars) {
    $logData = [
        'mailto' => $vars['mailto'],
        'subject' => $vars['subject'],
        'message_id' => $vars['message_id'] ?? null,
        'sent_at' => date('Y-m-d H:i:s'),
    ];
    
    Capsule::table('mod_email_logs')->insert($logData);
});
```

## Email Queue

```php
<?php
/**
 * Email queue processor
 */
function processEmailQueue(): int
{
    $pending = Capsule::table('mod_email_queue')
        ->where('status', 'pending')
        ->where('scheduled_at', '<=', date('Y-m-d H:i:s'))
        ->orderBy('priority', 'desc')
        ->orderBy('scheduled_at', 'asc')
        ->limit(100)
        ->get();
    
    $processed = 0;
    
    foreach ($pending as $item) {
        try {
            $email = json_decode($item->email_data, true);
            $email['to'] = [$item->recipient];
            
            $sendgrid = new SendGridEmailClient(SENDGRID_API_KEY);
            $result = $sendgrid->send($email);
            
            if ($result['success']) {
                Capsule::table('mod_email_queue')
                    ->where('id', $item->id)
                    ->update([
                        'status' => 'sent',
                        'sent_at' => date('Y-m-d H:i:s'),
                        'message_id' => $result['message_id'] ?? null,
                    ]);
                $processed++;
            } else {
                Capsule::table('mod_email_queue')
                    ->where('id', $item->id)
                    ->update([
                        'attempts' => $item->attempts + 1,
                        'last_error' => 'Send failed',
                    ]);
            }
            
        } catch (Exception $e) {
            Capsule::table('mod_email_queue')
                ->where('id', $item->id)
                ->update([
                    'attempts' => $item->attempts + 1,
                    'last_error' => $e->getMessage(),
                ]);
        }
    }
    
    return $processed;
}
```

## Best Practices

1. **Use email APIs** - Better deliverability than SMTP
2. **Implement bounces** - Handle bounced emails
3. **Track opens/clicks** - Monitor email engagement
4. **Warm up IPs** - Gradually increase volume
5. **Use templates** - Consistent branding
6. **Monitor spam scores** - Check before sending

## Related Documentation

- [whmcs-integration-api.md](whmcs-integration-api.md)
- [whmcs-integration-webhooks.md](whmcs-integration-webhooks.md)
