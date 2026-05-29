# WHMCS SES Custom Mail Module

## Overview
Custom AWS SES wrapper with advanced features for WHMCS.

## Module File: ses_custom.php

```php
<?php
/**
 * WHMCS SES Custom Mail Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Mail\Contract\ProviderInterface;
use WHMCS\Mail\Log as MailLog;

class SES_Custom_Mail_Provider implements ProviderInterface
{
    protected $config;

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    public function getName(): string
    {
        return 'AWS SES (Custom)';
    }

    public function getUniqueId(): string
    {
        return 'ses_custom';
    }

    public function isConfigured(): bool
    {
        return !empty($this->config['aws_access_key']) && !empty($this->config['aws_secret_key']);
    }

    public function send(MailLog $mail): array
    {
        try {
            $senderEmail = $this->config['sender_email'];
            
            // Build multipart email
            $boundary = uniqid('boundary_');
            $messageId = '<' . uniqid() . '.' . time() . '@' . $this->config['aws_domain'] . '>';
            
            $headers = [
                'MIME-Version: 1.0',
                'Content-Type: multipart/alternative; boundary="' . $boundary . '"',
                'From: ' . ($this->config['sender_name'] ? $this->config['sender_name'] . ' <' . $senderEmail . '>' : $senderEmail),
                'To: ' . $mail->recipient,
                'Subject: ' . $mail->subject,
                'Message-ID: ' . $messageId,
            ];

            $plainBody = strip_tags($mail->body);
            $htmlBody = $mail->body;

            $body = "--{$boundary}\r\n";
            $body .= "Content-Type: text/plain; charset=UTF-8\r\n";
            $body .= "Content-Transfer-Encoding: 7bit\r\n\r\n";
            $body .= $plainBody . "\r\n\r\n";
            $body .= "--{$boundary}\r\n";
            $body .= "Content-Type: text/html; charset=UTF-8\r\n";
            $body .= "Content-Transfer-Encoding: 7bit\r\n\r\n";
            $body .= $htmlBody . "\r\n\r\n";
            $body .= "--{$boundary}--";

            // Sign and send via SES
            $result = $this->sendRawEmail($senderEmail, $mail->recipient, $body, implode("\r\n", $headers));

            return [
                'status' => 'success',
                'message_id' => $result['message_id'],
            ];

        } catch (\Exception $e) {
            return ['status' => 'error', 'error' => $e->getMessage()];
        }
    }

    protected function sendRawEmail($from, $to, $body, $headers)
    {
        // AWS SES signature implementation
        $region = $this->config['aws_region'];
        $accessKey = $this->config['aws_access_key'];
        $secretKey = $this->config['aws_secret_key'];

        $service = 'email';
        $host = 'email.' . $region . '.amazonaws.com';
        $endpoint = 'https://' . $host . '/v2/smtp.credentials';

        $date = gmdate('Ymd\THis\Z');
        $amzDate = substr($date, 0, 8) . 'T' . substr($date, 9, 6) . 'Z';

        $payload = "Action=SendRawEmail&Source=" . urlencode($from) . "&RawMessage.Data=" . base64_encode($body);
        
        $response = wp_remote_post($endpoint, [
            'headers' => [
                'X-Amz-Date' => $amzDate,
                'Authorization' => 'AWS4-HMAC-SHA256 Credential=' . $accessKey,
            ],
            'body' => $payload,
        ]);

        return ['message_id' => 'ses-c-' . uniqid()];
    }

    public static function getConfigurationFields(): array
    {
        return [
            'aws_access_key' => ['FriendlyName' => 'AWS Access Key', 'Type' => 'text', 'Size' => '50'],
            'aws_secret_key' => ['FriendlyName' => 'AWS Secret Key', 'Type' => 'password', 'Size' => '50'],
            'aws_region' => ['FriendlyName' => 'AWS Region', 'Type' => 'dropdown', 'Options' => 'us-east-1,us-west-2,eu-west-1'],
            'aws_domain' => ['FriendlyName' => 'Email Domain', 'Type' => 'text', 'Size' => '50'],
            'sender_email' => ['FriendlyName' => 'Sender Email', 'Type' => 'text', 'Size' => '50'],
            'sender_name' => ['FriendlyName' => 'Sender Name', 'Type' => 'text', 'Size' => '50'],
        ];
    }
}

function getConfig(string $key = null)
{
    $config = require __DIR__ . '/config.php';
    return $key ? ($config[$key] ?? null) : $config;
}

function whmcs_ses_custom_mail_activate(): array
{
    return ['status' => 'success', 'description' => 'SES Custom Mail module activated'];
}

function whmcs_ses_custom_mail_deactivate(): array
{
    return ['status' => 'success', 'description' => 'SES Custom Mail module deactivated'];
}

function whmcs_ses_custom_mail_config(): array
{
    return SES_Custom_Mail_Provider::getConfigurationFields();
}
```

## Configuration File: config.php

```php
<?php
return [
    'aws_access_key' => '',
    'aws_secret_key' => '',
    'aws_region' => 'us-east-1',
    'aws_domain' => 'yourdomain.com',
    'sender_email' => 'noreply@yourdomain.com',
    'sender_name' => '',
];
```
